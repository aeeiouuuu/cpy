"""main.py

夕方サマリー自動生成アシスタントのメイン実行スクリプト。

流れ:
1. config/settings.yaml を読み込む
2. 各 collectors/*.py の collect() を呼び出して Item を集約する（天気・今日は何の日など）
   ※ ITmediaニュースはここには含めない（後述）
3. core/px_ai_client.py 経由でAI（PX-AI）にサマリー生成を依頼する
   - AI呼び出しに失敗した場合（エンドポイント未確認・ネットワーク不通など）は、
     Itemをカテゴリ別にそのまま並べる簡易フォーマットにフォールバックする
4. ITmediaニュースを、AIを経由せず固定で3件、末尾にそのまま追加する
   （旧Power Automateでのそのまま配信運用に合わせた形。見出し・本文はAIに要約させない）
5. out/draft_YYYYMMDD.md として下書きを書き出す

【注意】「📰 注目ニュース」カテゴリ（CATEGORY_LABELSの"news"）は、今後
internal_news.py（社内イントラ）を実装した際に使う想定に変更した。一般公開の
ニュース(Google News等)は、末尾に固定で追加するITmediaニュースでカバーする。
collectors/public_news.py 自体は削除していない（再利用したくなった場合のため）が、
現在main.pyからは呼んでいない。

【社用PCでのSSL証明書エラーについて】
社内プロキシ(WARP等)がTLS通信を検査する構成の場合、Pythonの`requests`は
社内ルート証明書を知らないため、天気・今日は何の日・ITmedia・PX-AIなど、
すべての外部通信がSSLエラーで失敗する（＝collector側のtry/exceptで握りつぶされ、
エラー表示なしに情報0件になる）ことが確認されている。対策として、Windowsが
信頼している証明書ストアをPythonにもそのまま使わせる`truststore`パッケージを
requestsより先にimportして有効化している（下記参照）。
"""

from __future__ import annotations

# 社内プロキシ等によるTLS通信の検査（証明書の差し替え）でSSL証明書エラーになる場合の対策。
# Windowsの証明書ストア（会社のルート証明書を含む）をPythonのssl検証にも使わせるようにする。
# requestsなど、ssl/httpsを使うモジュールをimportするより前に呼び出す必要がある。
import truststore

truststore.inject_into_ssl()

import argparse
import datetime as dt
from pathlib import Path

import yaml

from dotenv import load_dotenv

from collectors import (
    internal_birthday,
    internal_calendar,
    public_anniversary,
    public_itmedia,
    public_traffic,
    public_weather,
)
from collectors.base import Item
from core import px_ai_client

ROOT = Path(__file__).parent
CONFIG_DIR = ROOT / "config"
OUT_DIR = ROOT / "out"

load_dotenv(ROOT / ".env")  # .envがあれば環境変数として読み込む（無くてもエラーにはしない）

CATEGORY_LABELS = {
    "weather": "☀️ 天候と交通",
    "schedule": "🗓️ 明日の予定・期限",
    "celebration": "🎂 お祝い・その他",
    "news": "📰 注目ニュース",  # 今後internal_news.py(社内イントラ)向け。現状は使用するcollectorなし
}
CATEGORY_ORDER = ["weather", "schedule", "celebration", "news"]

ITMEDIA_HEADING = "### 📰 ITmediaニュース"


def load_settings() -> dict:
    settings_path = CONFIG_DIR / "settings.yaml"
    if not settings_path.exists():
        return {}
    with settings_path.open(encoding="utf-8") as f:
        return yaml.safe_load(f) or {}


def collect_all(settings: dict) -> list[Item]:
    """各collectorを呼び出してItemを集約する。

    collectors/base.py の方針どおり、collect()自体が内部でtry-exceptして
    失敗時は[]を返す実装になっている想定だが、念のためここでも保護する。

    TODO: internal_news.py を実装したらここに追記する（README.md 5章参照）。
          社内システム(Selenium等)へのアクセスが必要なため、社用PC・社内ネットワーク
          環境で別途実装する想定（このセッションでは未実装）。実装後は
          category="news" のItemを返すようにすれば、そのままAI要約の
          「📰 注目ニュース」に載る。

    ITmediaニュース（collectors/public_itmedia.py）はここには含めない。AIを経由せず
    固定で末尾に追加する別枠のため、run() から直接呼び出す。
    """
    items: list[Item] = []
    collectors = [
        ("public_weather", lambda: public_weather.collect(settings.get("weather"))),
        ("public_traffic", lambda: public_traffic.collect(settings.get("traffic"))),
        ("public_anniversary", lambda: public_anniversary.collect(settings.get("anniversary"))),
        ("internal_calendar", lambda: internal_calendar.collect(settings.get("schedule"))),
        ("internal_birthday", lambda: internal_birthday.collect(settings.get("birthday"))),
    ]
    for name, collect_fn in collectors:
        try:
            items.extend(collect_fn())
        except Exception as exc:  # collector側の実装漏れに対する保険
            print(f"[warn] collector '{name}' で想定外のエラー: {exc}")
    return items


def _format_item_text(item: Item) -> str:
    """Itemの表示テキストを組み立てる。urlがあればMarkdownハイパーリンクにする。

    - ニュースのように title と body が実質同じ内容の場合は、本文自体をリンクにする
      （例: "- [新型ノートPCが発表 - Example News](https://...)"）。
    - 天気のように body が長めの文章でtitleと異なる場合は、文末に「詳細」リンクを添える
      （例: "- 明日の大阪は「くもり」の予想です。...（[詳細](https://...)）"）。
    """
    if not item.url:
        return item.body
    if item.body.strip() == item.title.strip():
        return f"[{item.body}]({item.url})"
    return f"{item.body}（[詳細]({item.url})）"


def render_fallback_markdown(items: list[Item]) -> str:
    """AIを使わず、Itemをカテゴリ別にそのまま並べる簡易Markdown整形。

    AI連携がまだ設定できていない段階での動作確認や、AI呼び出し失敗時の
    フォールバックとして使う。
    """
    by_category: dict[str, list[Item]] = {}
    for item in items:
        by_category.setdefault(item.category, []).append(item)

    lines: list[str] = []
    rendered_categories = set()
    for category in CATEGORY_ORDER:
        cat_items = by_category.get(category, [])
        if not cat_items:
            continue
        rendered_categories.add(category)
        lines.append(f"### {CATEGORY_LABELS.get(category, category)}")
        for item in cat_items:
            lines.append(f"- {_format_item_text(item)}")
        lines.append("")

    # CATEGORY_ORDER に無いカテゴリも念のため出力する
    for category, cat_items in by_category.items():
        if category in rendered_categories:
            continue
        lines.append(f"### {category}")
        for item in cat_items:
            lines.append(f"- {_format_item_text(item)}")
        lines.append("")

    if not lines:
        return "(本日は収集できた情報がありませんでした)\n"
    return "\n".join(lines).strip() + "\n"


def render_itmedia_block(itmedia_items: list[Item]) -> str:
    """ITmediaニュースを、AIを介さずそのままMarkdownの箇条書きブロックにする。

    旧Power Automateでの「RSSフィードをそのまま配信」という運用・見た目に合わせるため、
    要約や言い換えはせず、タイトルをそのままハイパーリンクにして並べるだけにしてある。
    取得できなかった場合（ネットワーク不通等）は空文字列を返す（＝末尾に何も追加しない）。
    """
    if not itmedia_items:
        return ""
    lines = [ITMEDIA_HEADING]
    for item in itmedia_items:
        lines.append(f"- {_format_item_text(item)}")
    return "\n".join(lines) + "\n"


def build_prompt(items: list[Item], settings: dict) -> str:
    """config/policy.md の方針とItem一覧から、PX-AIに渡すプロンプト文字列を組み立てる。

    AI呼び出し本体（chat()）とは分離してあるので、実際に送信する前に
    この関数の戻り値だけを確認する（= --dry-run）ことができる。
    """
    policy_path = CONFIG_DIR / "policy.md"
    policy_text = policy_path.read_text(encoding="utf-8") if policy_path.exists() else ""

    items_text = (
        "\n".join(f"- [{item.category}] {_format_item_text(item)}" for item in items)
        or "(収集できた情報はありません)"
    )
    return (
        f"{policy_text}\n\n"
        "---\n"
        "以下は本日収集した情報です。上記の方針に沿って、社内チャット投稿用のサマリーを"
        "Markdown形式で作成してください。各項目に既にMarkdownのハイパーリンク（[表示名](URL)の形式）が"
        "含まれている場合は、生成するサマリーでもそのリンクをそのまま保持してください。\n\n"
        f"{items_text}"
    )


def render_with_ai(items: list[Item], settings: dict) -> str:
    """config/policy.md の方針とItem一覧をAIに渡し、サマリーMarkdownを生成する。"""
    prompt = build_prompt(items, settings)
    return px_ai_client.chat(
        [{"role": "user", "content": prompt}],
        config=settings.get("ai"),
    )


def run(dry_run: bool = False) -> Path:
    settings = load_settings()
    items = collect_all(settings)
    itmedia_items = public_itmedia.collect(settings.get("itmedia"))
    itmedia_block = render_itmedia_block(itmedia_items)

    OUT_DIR.mkdir(exist_ok=True)
    today_str = dt.date.today().strftime("%Y%m%d")

    if dry_run:
        # PX-AIには送信せず、送信予定のプロンプト文面だけを確認する。
        # policy.md の内容やcollectorの収集結果が意図通りかを、AI呼び出し
        # （APIキー・ネットワーク疎通が必要）に先立って確認できる。
        # ITmediaニュースはAIに渡さない別枠なので、参考として別途表示する
        # （こちらはネットワークさえ繋がれば今すぐ確認できる）。
        prompt = build_prompt(items, settings)
        prompt_path = OUT_DIR / f"prompt_{today_str}.md"
        prompt_path.write_text(prompt, encoding="utf-8")
        print("=== PX-AIに送信予定のプロンプト（--dry-run のため実際には送信していません） ===\n")
        print(prompt)
        print(f"\n=== ここまで（{prompt_path} にも保存しました） ===")
        print("\n=== ITmediaニュース（AIには渡さず、常に末尾に固定でこのまま追加されます） ===\n")
        print(itmedia_block if itmedia_block else "(取得できませんでした)")
        return prompt_path

    try:
        markdown = render_with_ai(items, settings)
    except Exception as exc:
        # policy.md がまだ雛形のままだったり、PX-AIのエンドポイントが未確認だったりする
        # 現段階では失敗して当然のため、簡易フォーマットにフォールバックして動作は止めない。
        print(f"[warn] AI要約に失敗したため、簡易フォーマットで出力します: {exc}")
        markdown = render_fallback_markdown(items)

    if itmedia_block:
        markdown = markdown.rstrip() + "\n\n" + itmedia_block

    out_path = OUT_DIR / f"draft_{today_str}.md"
    out_path.write_text(markdown, encoding="utf-8")
    print(f"draftを書き出しました: {out_path}")
    return out_path


if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="夕方サマリー自動生成アシスタント")
    parser.add_argument(
        "--dry-run",
        action="store_true",
        help="PX-AIには送信せず、送信予定のプロンプト内容だけを確認する",
    )
    args = parser.parse_args()
    run(dry_run=args.dry_run)
