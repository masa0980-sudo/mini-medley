# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

手持ちのブラウザゲームを**十数秒ずつ連続で遊び、最後に1つの総合スコア**を出すハブ（リズム天国・メイドインワリオ型）。
ゲームは別々のリポジトリ・別々の公開ページ（`masa0980-sudo.github.io/<リポ名>/`）なので、ハブは
**iframe を1本ずつ作って呼び出し、`postMessage` で成績を受け取る**だけ。ゲーム本体は持たない。

姉妹リポジトリ（`mochi-exit` / `mochi-jump` / `tennis-game` など）と同じ構成: **`index.html` 一枚、ビルド無し、
依存なし、GitHub Pages**。ツールを足さないこと。

## Commands

- ローカル実行: `python -m http.server 8480` → `http://localhost:8480/index.html`
- 検証: Playwright headless。自動テスト基盤は無い。`?fast=1`（演出を短縮）と `?ids=` / `?u.<id>=` を使う（下記）
- デプロイ: `main` に push で `.github/workflows/deploy-pages.yml` が Pages を更新。初回は
  `gh api -X POST repos/<owner>/<repo>/pages -f build_type=workflow` で Pages サイトを先に作る

## ミニ版プロトコル（ゲーム側との約束・これが仕様）

**ハブ → ゲーム**: URL に `?mini=1&sec=<秒>&mute=<0|1>` を付けて iframe で開く。
ゲームは `mini=1` のとき、次をすべて守る。`mini=1` が無いときは**今までと完全に同じ動き**にする。

1. タイトル・選択・あそびかたを出さず、**待機状態**で止まる（BGM・タイマーも始めない）。
2. 準備ができたら `parent.postMessage({type:"mini-ready", game:"<id>"}, "*")`。
3. ハブから `{type:"mini-go"}` が届いたら**そこで初めて**開始する（カウントダウンはハブが出す）。
4. `sec` 秒で必ず打ち切る（ゲームオーバー・クリアで早く終わってもよい）。
5. 終了時に**必ず1回**、`parent.postMessage({type:"mini-result", game:"<id>", score:<0〜100>, detail:{…}}, "*")`。
   **0〜100への換算はゲーム側が持つ**（内部事情を知っているのはゲーム側だけ。ハブは足してランクを付けるだけ）。
6. `PlayCounts`（Firestore）と自己ベストは**書かない**。メドレー1回につき `mini-medley` をハブが1回数える。
7. `mute=1` なら音を出さない。BGM・効果音は1つのマスターゲイン経由にして 0 にできるようにしておく。
8. 結果画面・「もういちど」・タイトルへ戻る等の余計なUIは出さない（終わったら結果を返して静かに待つ）。

**ハブ側の受け取り**: `event.source` が今の iframe の `contentWindow` で、`d.game` がそのゲームの id のものだけを受け入れる
（他のメッセージは無視。偽メッセージのテスト済み）。`score` は 0〜100 に丸めて・範囲に収める（数値でなければ 0）。
`mini-ready` が `T.ready`（既定9秒）来なければ「読み込めませんでした（0点）」、
`mini-result` が `sec + T.grace`（既定8秒）来なければ 0 点で先へ進む。

**iframe は常に1つ**: 1本終わるたびに `remove()` してから次を作る。残すと、そのゲームの音もタイマーも
鳴り続ける（`display:none` にしても `document.hidden` は変わらないので、ゲーム側の自動ポーズも効かない）。

## Architecture（`index.html` 内の番号付きモジュール）

1. **GAMES** — 呼び出すゲームの表。`ready:false` のゲームは抽選に入らない（ゲーム側が `?mini=1` に対応するまで）。
   `?ids=a,b,c` で出すゲームと順番を固定（`ready` 無視・テスト用）、`?u.<id>=<url>` で URL を差し替え
   （ローカルのゲームを呼ぶ用）。**公開は必ずゲーム側を先に push してから `ready:true` にする**。
   **全ゲームを一覧に載せる方針**（2026-10-31）: ミニ版がまだ無いゲーム（ぷよぷよ・テトリス・名言タイピング）も `ready:false` で
   載せてあり、設定画面に「準備中」の灰色で出る。ゲームを足すのは `GAMES` に1行＋`DESC` に説明1行だけ。
   **せってい（`#settings`）**: タイトルの「せってい」から開き、遊べる（`ready:true`）ゲームごとのチェックで抽選に入れる／外す。
   保存は `localStorage["mini-medley:disabled"]`（**外したゲームの id だけ**を持つので、あとで足したゲームは最初から選ばれる）。
   `chosenPool()` が抽選の元で、選んだゲームが3本以下ならその全部を遊ぶ・0本なら「えらんだゲームがありません」と出して始めない。
   `?ids=` で固定したときは設定を無視する（テスト用）。
2. **Sfx** — カウントダウン・得点の音だけ。`master` ゲインを通し、ミュートは `localStorage["mini-medley:muted"]`。
3. **PlayCounts** — Firestore REST。ローカル配信では書かない。`mochi-exit` からそのまま移した。
4. **Best** — `localStorage["mini-medley:best"]` に `{total,max}`。本数が違うメドレーでも比べられるよう
   **満点に対する割合**で更新を判定する。
5. **進行** — `runMedley(list)` が `playOne` を順に呼ぶ。待ちは `sleep`/`waitMsg`（どちらも `cancels` に登録され、
   `quit()` の `cancelAll()` で即解ける）。**`playOne` は冒頭で `const me = run` と控える**: `quit()` が
   `run = null` にした後に `run.abort` を読むと TypeError になる（実際に踏んだ）。
6. 画面のつなぎ・共有（`navigator.share` → クリップボード）・テストフック `window.__medley`。

ランクは合計の満点に対する割合: S ≥ 85% / A ≥ 70% / B ≥ 50% / それ以外 C。
タイトルは**ロゴ+はじめる+あそびかた+せってい**だけ（ルール文は `#howtoOverlay`。ユーザーの固定方針）。

## `window.__medley` — テストフック

`games`（id・ready・sec の一覧）/ `state()`（進行中の idx・total・results・iframe 数。無ければ null）/ `frames()` /
`shareText()` / `isMuted()` / `start()` / `pool()`（いま抽選に入るゲームの id）/ `setDisabled(ids)`。`test/dummy.html` はプロトコルを最小で実装したダミー
（`?score=` で点を指定、`?noready=1` / `?noresult=1` で異常系）。通しテストは
`?fast=1&ids=dummy,dummy,dummy&u.dummy=./test/dummy.html%3Fscore%3D80` で開き `#startBtn` → `#result` が active になるまで待つ。

## Scope

3本前後を連続で遊ぶ1モードのみ。遊ぶゲームの取捨選択はできる（せってい）。ランキング（Firestore 書き込み）・
1回の本数（3本固定）・難易度は対象外（要望が出てから）。

## 現状（2026-10-28 公開・10-30 neon-void・Rhythm_game・クイズ2本を追加）

公開URL: https://masa0980-sudo.github.io/mini-medley/ 。`ready:true` は mochi-exit / mochi-jump / tennis-game / neon-void /
rhythm-game / daily-claude-quiz / daily-genai-quiz の7本（各ゲーム側のミニ版は先に公開済み）。1回のメドレーは7本からランダムに3本。
公開中の本物で、ボットによる通しプレイ
（ランダム順・iframe は常に1つ・各ゲーム開始時に AudioContext が `running`）を確認した。

| ゲーム | ミニ版の内容 | 100点の目安 | 検証で測った値 |
|---|---|---|---|
| mochi-exit | **30秒**（覚える5秒＋遊ぶ25秒）で出口8まで（正解−ミス）。難度3の異変は出さない | 無ミスで出口8クリア | 2.2秒に1回しか操作できないボットでも全問正解で100（約22秒） |
| mochi-jump | ステージ1の距離75%＋きなこ25%。ゴールで100 | ゴール到達 | 何もしないと1、x=2000地点で40 |
| tennis-game | ラリー得点 ÷ 1500 | 完璧なタイミング | 完璧ボットで20秒に9回・1620点 |
| neon-void | ステージ1・NORMAL。スコア ÷ 2000 | 約2000点（コンボを繋いで撃破） | 撃つだけのボットは20秒で500〜850点（25〜40点） |
| rhythm-game | ぴったりリング(ノーマル)24拍。判定点 ÷ 2400（PERFECT 100・GOOD 50） | 全部PERFECT | 完璧ボット92〜100・90msずれ(全GOOD)50・無入力0。約18秒で自然に終わる |
| daily-claude-quiz / daily-genai-quiz | **`sec=30`**。ランダムな3問。1問ごとに制限時間（約9.1秒）・正解で20〜33.3点（速いほど高い） | 3問とも即答で正解 | 2秒ずつで全問正解91・4.5秒80・7秒69・9.5秒かけると時間切れで0。`sec` ぴったりかかる |

**「短すぎて遊べない」と指摘されて調整した（2026-10-31）**: もち出口は覚える時間2.2秒・20秒・難度3まで出題、クイズは1問5.8秒
（問題文と4択を読むには足りない）だった。ゲームごとの長さは `GAMES` の `sec` で変える（ゲーム側は `sec` から自分で
時間配分を計算する）。**新しいゲームを足すときは「初見の人が読む・覚える時間」を先に見積もってから `sec` を決める**。

**点数の目安は人が遊んで調整する**（上の値はボットの上限・下限で、普通の人の点は未測定）。
実機（スマホ）での音の自動再生とタッチ操作は未確認。

注意: rhythm-game は音ズレ調整（`rhythm-ring-offset`）を端末に保存済みの値で使う。ハブと Rhythm_game は同じ
`masa0980-sudo.github.io` のオリジンなので、前に Rhythm_game 本体で計測していればそのまま効く（未計測なら 0）。
自然な長さが約18秒なので、`sec` を18秒以下にするとゲーム側が途中で打ち切って点が下がる。

クイズ2本は**毎日の3問ではなくランダムな3問**を出す（同じ日に何度遊んでも別の問題）。ミニ版では解説を出さない
（学習用ではなく反応の速さを競う形にしたため）。

次の候補: お題の本数や順番の選択、ランキング、実機での調整。ゲームを足す手順: ①ゲーム側に `?mini=1` を実装して push →
②`GAMES` に1行足して `ready:true` → ③上のプロトコルの項目を1つずつ確認。
