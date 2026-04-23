<script>
  import { supabase } from "$lib/supabase.js";
  import { onMount } from "svelte";

  // 各モードの件数
  let unansweredCount = $state(0);
  let reviewCount = $state(0);
  let weakCount = $state(0);

  // 読み込み中かどうか
  let loading = $state(true);

  // 未回答の件数を取得する関数
  async function fetchUnansweredCount() {
    // ① 回答済みのquestion_idを取得する
    const { data: progressData, error: progressError } = await supabase.from("kanji_progress").select("question_id");

    if (progressError) {
      console.error("進捗取得エラー:", progressError);
      return;
    }

    const answeredIds = progressData.map((p) => p.question_id);

    if (answeredIds.length === 0) {
      // 一度も回答していない場合は全件数を取得する
      const { count, error } = await supabase.from("kanji_questions").select("*", { count: "exact", head: true });

      if (error) {
        console.error("件数取得エラー:", error);
        return;
      }
      unansweredCount = count ?? 0;
    } else {
      // 回答済みIDを除いた件数を取得する
      const { count, error } = await supabase
        .from("kanji_questions")
        .select("*", { count: "exact", head: true })
        .not("id", "in", `(${answeredIds.join(",")})`);

      if (error) {
        console.error("件数取得エラー:", error);
        return;
      }
      unansweredCount = count ?? 0;
    }
  }

  // 復習の件数を取得する関数
  async function fetchReviewCount() {
    const today = new Date().toISOString().split("T")[0];

    const { count, error } = await supabase.from("kanji_progress").select("*", { count: "exact", head: true }).lte("next_review_at", today);

    if (error) {
      console.error("復習件数取得エラー:", error);
      return;
    }

    reviewCount = count ?? 0;
  }

  // 苦手一覧の件数を取得する関数
  async function fetchWeakCount() {
    const { count, error } = await supabase.from("kanji_progress").select("*", { count: "exact", head: true }).gt("incorrect_count", 0);

    if (error) {
      console.error("苦手件数取得エラー:", error);
      return;
    }

    weakCount = count ?? 0;
  }

  // ページ表示時に全件数を取得する
  onMount(async () => {
    await Promise.all([fetchUnansweredCount(), fetchReviewCount(), fetchWeakCount()]);
    loading = false;
  });
</script>

<div class="menu">
  <h1>漢字練習アプリ</h1>

  {#if loading}
    <p class="loading">読み込み中...</p>
  {:else}
    <!-- 学習メニュー -->
    <div class="cards">
      <!-- 未回答 -->

      <a
        href="/card?mode=unanswered"
        class="card"
        class:disabled={unansweredCount === 0}
        aria-disabled={unansweredCount === 0}
        onclick={(e) => {
          if (unansweredCount === 0) e.preventDefault();
        }}
      >
        <span class="icon">📝</span>
        <span class="label">未回答</span>
        <span class="count">{unansweredCount}件</span>
      </a>

      <!-- 復習 -->

      <a
        href="/card?mode=review"
        class="card"
        class:disabled={reviewCount === 0}
        aria-disabled={reviewCount === 0}
        onclick={(e) => {
          if (reviewCount === 0) e.preventDefault();
        }}
      >
        <span class="icon">🔁</span>
        <span class="label">復習</span>
        <span class="count">{reviewCount}件</span>
      </a>

      <!-- 苦手一覧 -->

      <a
        href="/card?mode=weak"
        class="card"
        class:disabled={weakCount === 0}
        aria-disabled={weakCount === 0}
        onclick={(e) => {
          if (weakCount === 0) e.preventDefault();
        }}
      >
        <span class="icon">😓</span>
        <span class="label">苦手一覧</span>
        <span class="count">{weakCount}件</span>
      </a>
    </div>

    <!-- 問題登録リンク -->
    <a href="/register" class="register-link">⚙️ 問題を登録する</a>
  {/if}
</div>

<style>
  /* メニュー全体 */
  .menu {
    max-width: 480px;
    margin: 40px auto;
    padding: 24px;
    font-family: sans-serif;
    text-align: center;
  }

  h1 {
    font-size: 24px;
    margin-bottom: 32px;
  }

  /* 読み込み中 */
  .loading {
    color: #999;
  }

  /* カードのグリッド */
  .cards {
    display: flex;
    flex-direction: column;
    gap: 12px;
    margin-bottom: 32px;
  }

  /* 各カード */
  .card {
    display: flex;
    align-items: center;
    padding: 16px 20px;
    border: 1px solid #ddd;
    border-radius: 12px;
    text-decoration: none;
    color: #333;
    background: #fff;
    transition: background 0.15s;
  }

  .card:hover:not(.disabled) {
    background: #f5f5f5;
  }

  /* グレーアウト（0件のとき） */
  .card.disabled {
    background: #f9f9f9;
    color: #bbb;
    cursor: not-allowed;
    border-color: #eee;
  }

  /* アイコン */
  .icon {
    font-size: 24px;
    margin-right: 12px;
  }

  /* ラベル */
  .label {
    font-size: 18px;
    flex: 1;
    text-align: left;
  }

  /* 件数 */
  .count {
    font-size: 16px;
    color: #888;
  }

  .card.disabled .count {
    color: #ccc;
  }

  /* 問題登録リンク */
  .register-link {
    display: inline-block;
    margin-top: 8px;
    font-size: 18px;
    color: #888;
    text-decoration: none;
  }

  .register-link:hover {
    color: #333;
  }
</style>
