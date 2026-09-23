"""collectors/public_traffic.py

京橋オフィス通勤者向けの交通・気象警報情報を2種類取得する。category="weather"として、
既存の「☀️ 天候と交通」枠にそのまま統合する（public_weather.pyの天気予報と合わせて表示される）。

1. 気象庁 警報・注意報（台風・大雨・大雪など、在宅勤務/出社判断に関わるもの）
   - API: https://www.jma.go.jp/bosai/warning/data/warning/{area_code}.json
   - 認証不要・無料の官公庁公開データ。public_weather.pyと同じ大阪府コード(270000)を既定値とする
     （気象庁APIは市区町村単位の粒度を提供していないため）。
   - warnings配列の中から、台風・大雪関連の警報コードのみを対象に、statusが「解除」
     「発表警報・注意報はなし」以外（＝現在発表中）のものを拾う。

2. JR西日本 運行情報（大阪環状線・JR東西線）
   - API: https://trafficinfo.westjr.co.jp/api/v1/trafficinfo.json
   - JR西日本の公式サイト(trafficinfo.westjr.co.jp)が自身のページ描画に使っている
     一般公開JSON（認証不要）。dailyDataに当日を含め数日分（今日・明日・明後日）の
     情報が入っており、明日分の計画運休・工事等の告知があれば拾える。
     ただし突発的な当日朝の遅延・事故等は当然ながら前日時点では予測できない
     （このcollectorが拾えるのは「あらかじめ判明している」情報のみ）。
   - 平常運転の路線はJSON内に一切登場しない、という設計を確認済み（該当が無ければ
     何も出力しない＝他のcollectorと同じ方針）。
   - 【利用規約について】trafficinfo.westjr.co.jpの利用規約には「本サービスを個人的な
     利用範囲を超えて、許可なく商業・営利目的において利用する行為」「当社に無断で複製・
     送信等を行う行為」を禁止する条項がある。本ツールは社内の少人数チーム向け、非営利の
     情報共有目的であることを前提に、利用者の判断で有効化している
     （config/settings.yaml の traffic.jr_west.enabled で無効化可能）。

【設計メモ】このファイルは2つの独立した外部APIを叩くため、他のcollectorと異なり
ソースごとに個別のtry/exceptで保護している（片方のAPIが落ちていても、もう片方の
結果は失わないようにするため）。最終的にcollect()全体は失敗しても[]を返す。
"""

from __future__ import annotations

import datetime as dt
from typing import Optional

import requests

from .base import Item

REQUEST_TIMEOUT_SECONDS = 10

# --- 気象庁 警報・注意報 ---

JMA_WARNING_URL = "https://www.jma.go.jp/bosai/warning/data/warning/{area_code}.json"
DEFAULT_JMA_AREA_CODE = "270000"  # 大阪府（public_weather.pyと同じ粒度）

# 現在発表中とみなさないstatus値（このいずれか以外なら「発表中」として扱う）
_INACTIVE_STATUSES = {"解除", "発表警報・注意報はなし"}

# 台風・大雨・大雪など、在宅勤務/出社判断に関わる警報・注意報コードのみを対象にする
# （気象庁防災情報XMLフォーマットのコード管理表に基づく。全種類ではなく抜粋）
_RELEVANT_WARNING_CODES = {
    "02": "暴風雪警報",
    "03": "大雨警報",
    "05": "暴風警報",
    "06": "大雪警報",
    "07": "波浪警報",
    "08": "高潮警報",
    "32": "暴風雪特別警報",
    "33": "大雨特別警報",
    "35": "暴風特別警報",
    "36": "大雪特別警報",
    "37": "波浪特別警報",
    "38": "高潮特別警報",
    "12": "大雪注意報",
    "13": "風雪注意報",
    "15": "強風注意報",
}


def _collect_jma_warnings(config: dict) -> list[Item]:
    area_code = config.get("jma_area_code", DEFAULT_JMA_AREA_CODE)
    url = JMA_WARNING_URL.format(area_code=area_code)

    response = requests.get(url, timeout=REQUEST_TIMEOUT_SECONDS)
    response.raise_for_status()
    data = response.json()

    # areaTypes[0] が都道府県単位（area_codeそのもの）を含む想定
    area_types = data.get("areaTypes") or []
    if not area_types:
        return []
    areas = area_types[0].get("areas") or []

    active_names: list[str] = []
    for area in areas:
        if area.get("code") != area_code:
            continue
        for warning in area.get("warnings") or []:
            status = warning.get("status")
            if status in _INACTIVE_STATUSES:
                continue
            name = _RELEVANT_WARNING_CODES.get(warning.get("code"))
            if name and name not in active_names:
                active_names.append(name)

    if not active_names:
        return []

    names_text = "・".join(active_names)
    return [
        Item(
            category="weather",
            title="気象警報・注意報",
            body=f"{names_text}が発表中です。台風・大雪等の場合は在宅勤務や出社時刻の見直しも検討しましょう。",
            source="public_traffic_jma",
            url="https://www.jma.go.jp/bosai/warning/",
            priority=15,  # 通常の天気予報(priority=10)より優先度高め（緊急性が高いため）
            raw={"active_warnings": active_names},
        )
    ]


# --- JR西日本 運行情報（大阪環状線・JR東西線） ---

JR_WEST_TRAFFICINFO_URL = "https://trafficinfo.westjr.co.jp/api/v1/trafficinfo.json"
DEFAULT_JR_WEST_TARGET_LINES = ["大阪環状線", "ＪＲ東西線"]


def _tomorrow_jst() -> dt.date:
    jst = dt.timezone(dt.timedelta(hours=9))
    return (dt.datetime.now(jst) + dt.timedelta(days=1)).date()


def _collect_jr_west(config: dict) -> list[Item]:
    if not config.get("enabled", True):
        return []

    target_lines = config.get("target_lines", DEFAULT_JR_WEST_TARGET_LINES)
    tomorrow_str = _tomorrow_jst().isoformat()

    response = requests.get(JR_WEST_TRAFFICINFO_URL, timeout=REQUEST_TIMEOUT_SECONDS)
    response.raise_for_status()
    data = response.json()

    items: list[Item] = []
    for area in data.get("areaTrafficInfos") or []:
        for daily in area.get("dailyData") or []:
            if daily.get("date") != tomorrow_str:
                continue
            for place in daily.get("placeTrafficInfos") or []:
                for line in place.get("conventionalLineTrafficInfos") or []:
                    line_name = line.get("lineName")
                    if line_name not in target_lines:
                        continue
                    for detail in line.get("conventionalLineTrafficInfoDetails") or []:
                        items.append(_build_jr_west_item(line_name, detail))
    return items


def _build_jr_west_item(line_name: str, detail: dict) -> Item:
    condition = detail.get("conditionName") or "運行情報あり"
    cause = detail.get("cause")

    section_texts = [
        f"{sec.get('startStation')}〜{sec.get('endStation')}駅間"
        for sec in (detail.get("sections") or [])
        if sec.get("startStation") and sec.get("endStation")
    ]

    detail_bits = []
    if section_texts:
        detail_bits.append("・".join(section_texts))
    if cause:
        detail_bits.append(f"原因: {cause}")

    body = f"明日、{line_name}で「{condition}」の情報があります"
    if detail_bits:
        body += "（" + "、".join(detail_bits) + "）"
    body += "。"

    return Item(
        category="weather",
        title=f"{line_name} 運行情報",
        body=body,
        source="public_traffic_jrwest",
        url="https://trafficinfo.westjr.co.jp/kinki.html",
        priority=12,
        raw=detail,
    )


def collect(config: Optional[dict] = None) -> list[Item]:
    """気象警報・注意報とJR西日本運行情報をItemとして返す（該当なしなら空リスト）。

    config: config/settings.yaml の `traffic` セクションを想定。
        例: {"jma_area_code": "270000", "jr_west": {"enabled": true, "target_lines": [...]}}

    JMA・JR西日本それぞれ個別にtry/exceptで保護しているため、片方が失敗しても
    もう片方の結果は返す。両方失敗、またはconfig自体が無い場合は空リストを返す。
    """
    config = config or {}
    items: list[Item] = []

    try:
        items.extend(_collect_jma_warnings(config))
    except Exception:
        pass

    try:
        items.extend(_collect_jr_west(config.get("jr_west") or {}))
    except Exception:
        pass

    return items


if __name__ == "__main__":
    # 動作確認用: python -m collectors.public_traffic
    for item in collect():
        print(item)
