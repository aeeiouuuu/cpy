"""collectors/internal_birthday.py

config/members.yaml の `members` セクションから、メンバーの誕生日お祝いの
お知らせを「1週間前」「前日」の2タイミングでItem化する。外部API不要。

- 対象は `consent: true` のメンバーのみ（本人の同意が無い人は除外する）。
- `birthday_md`（"MM-DD"形式）と、明日の日付を比較して判定する:
    - 前日リマインド: 明日がちょうど誕生日（＝今日案内すれば「明日は誕生日」と言える）
    - 1週間前リマインド: 明日から7日後が誕生日
  どのタイミングで知らせるかは config/settings.yaml の `birthday.remind_days_before`
  （日数のリスト）で変更可能。既定値は [7, 1]（1週間前・前日）。

【注意】2/29生まれのメンバーは、うるう年でない年はその日付自体が存在しないため、
該当年はリマインドが発生しない（簡易的な仕様上の制約）。
"""

from __future__ import annotations

import datetime as dt
from pathlib import Path
from typing import Optional

import yaml

from .base import Item

ROOT = Path(__file__).parent.parent
DEFAULT_MEMBERS_PATH = ROOT / "config" / "members.yaml"
DEFAULT_REMIND_DAYS_BEFORE = [7, 1]


def _tomorrow_jst() -> dt.date:
    jst = dt.timezone(dt.timedelta(hours=9))
    return (dt.datetime.now(jst) + dt.timedelta(days=1)).date()


def _load_yaml(path: Path) -> dict:
    if not path.exists():
        return {}
    with path.open(encoding="utf-8") as f:
        return yaml.safe_load(f) or {}


def _message_for_offset(display_name: str, offset: int, month: int, day: int) -> str:
    if offset == 1:
        return f"明日は{display_name}さんの誕生日です！"
    if offset == 7:
        return f"来週{month}/{day}は{display_name}さんの誕生日です！"
    # 7・1以外のオフセットを設定した場合の汎用メッセージ
    return f"{offset}日後（{month}/{day}）は{display_name}さんの誕生日です！"


def collect(config: Optional[dict] = None) -> list[Item]:
    """明日基準で、メンバーの誕生日お祝いリマインドをItemとして返す。

    config: config/settings.yaml の `birthday` セクションを想定（任意）。
        例: {"members_path": "config/members.yaml", "remind_days_before": [7, 1]}

    エラー時（YAML未整備・想定外の形式など）は空リストを返し、全体を止めない。
    """
    members_path = DEFAULT_MEMBERS_PATH
    remind_days_before = DEFAULT_REMIND_DAYS_BEFORE
    if config:
        if config.get("members_path"):
            members_path = ROOT / config["members_path"]
        if config.get("remind_days_before"):
            remind_days_before = config["remind_days_before"]

    try:
        tomorrow = _tomorrow_jst()
        data = _load_yaml(members_path)
        members = data.get("members") or []

        items: list[Item] = []
        for member in members:
            member = member or {}
            if not member.get("consent"):
                continue
            display_name = member.get("display_name")
            birthday_md = member.get("birthday_md")
            if not display_name or not birthday_md:
                continue

            for offset in remind_days_before:
                target = tomorrow + dt.timedelta(days=offset)
                if target.strftime("%m-%d") != birthday_md:
                    continue
                priority = 6 if offset == 1 else 2  # 前日の方を1週間前より少し優先度高めに
                items.append(
                    Item(
                        category="celebration",
                        title=f"{display_name}さんの誕生日",
                        body=_message_for_offset(display_name, offset, target.month, target.day),
                        source="internal_birthday",
                        url=None,
                        priority=priority,
                        raw={"member": display_name, "offset_days": offset, "target_date": str(target)},
                    )
                )

        return items
    except Exception:
        # README記載のルールに従い、失敗時は空リストを返して全体を止めない
        return []


if __name__ == "__main__":
    # 動作確認用: python -m collectors.internal_birthday
    for item in collect():
        print(item)
