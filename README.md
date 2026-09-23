# cpy
"""debug_network.py

evening-summary の各collectorが通信している外部サイトに、実際に到達できるかを
1つずつ確認するための診断用スクリプト（本体のcollectorとは独立して動く）。

collectors/*.py は失敗時に例外を握りつぶして [] を返す設計になっているため、
「エラーは出ないのに情報が集まらない」場合、このスクリプトで直接原因（タイムアウト・
SSL証明書エラー・プロキシ認証エラーなど）を確認する。

使い方:
    python debug_network.py

evening-summary プロジェクトのフォルダ内で実行してください
（requests がインストール済みであること）。
"""

import os
import sys
import traceback

import requests

TARGETS = [
    ("気象庁(JMA) 天気予報", "https://www.jma.go.jp/bosai/forecast/data/forecast/270000.json"),
    ("今日は何の日(whatistoday.cyou)", "https://api.whatistoday.cyou/v3/anniv/0924"),
    ("ITmedia RSS", "https://rss.itmedia.co.jp/rss/2.0/news_bursts.xml"),
]


def check_env():
    print("=" * 60)
    print("[環境情報]")
    print("Python:", sys.version.split()[0])
    print("requests:", requests.__version__)
    for key in ("HTTP_PROXY", "HTTPS_PROXY", "http_proxy", "https_proxy", "NO_PROXY", "REQUESTS_CA_BUNDLE", "SSL_CERT_FILE"):
        val = os.environ.get(key)
        if val:
            print(f"環境変数 {key} = {val}")
    print("=" * 60)


def check_target(name: str, url: str) -> None:
    print(f"\n--- {name} ---")
    print(f"URL: {url}")
    try:
        resp = requests.get(
            url,
            headers={"User-Agent": "evening-summary-debug/1.0"},
            timeout=10,
        )
        resp.raise_for_status()
        print(f"[OK] status={resp.status_code} bytes={len(resp.content)}")
    except requests.exceptions.SSLError:
        print("[NG] SSLエラー（証明書検証に失敗）")
        print("     → 社内プロキシ/WARP等によるTLS通信の検査（MITM）で、")
        print("       Pythonが社内ルート証明書を信頼できていない可能性が高いです。")
        traceback.print_exc()
    except requests.exceptions.ProxyError:
        print("[NG] プロキシエラー")
        print("     → 社内プロキシの認証が必要、またはプロキシ設定がPython側に")
        print("       渡っていない可能性があります。")
        traceback.print_exc()
    except requests.exceptions.ConnectTimeout:
        print("[NG] 接続タイムアウト")
        print("     → 通信自体がブロックされている（ファイアウォール等）可能性があります。")
        traceback.print_exc()
    except requests.exceptions.ConnectionError:
        print("[NG] 接続エラー")
        traceback.print_exc()
    except Exception:
        print("[NG] その他のエラー")
        traceback.print_exc()


if __name__ == "__main__":
    check_env()
    for name, url in TARGETS:
        check_target(name, url)
    print("\n" + "=" * 60)
    print("上記の結果を教えてください。[OK]がどれもゼロ件なら通信そのものがブロック、")
    print("SSLエラーが出ていれば社内プロキシのTLS検査が原因の可能性が高いです。")
