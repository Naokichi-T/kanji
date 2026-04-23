<script>
  import { supabase } from "$lib/supabase.js";
  import { onMount } from "svelte";
  import { page } from "$app/stores";
  import { get } from "svelte/store";

  // URLパラメータからmodeを取得する（デフォルトはunanswered）
  let mode = $state("unanswered");

  // カードのリスト
  let questions = $state([]);

  // 今何番目のカードを表示しているか（0始まり）
  let currentIndex = $state(0);

  // 答えを表示しているかどうか
  let showAnswer = $state(false);

  // データ取得中かどうか
  let loading = $state(true);

  // 現在のカード
  let currentQuestion = $derived(questions[currentIndex]);

  // 完了したかどうか
  let isCompleted = $derived(currentIndex >= questions.length && questions.length > 0);

  // Supabaseから問題を全件取得する関数
  // Supabaseから問題を全件取得する関数
  async function fetchQuestions() {
    // URLパラメータからmodeを取得する
    mode = get(page).url.searchParams.get("mode") ?? "unanswered";

    let result;

    if (mode === "unanswered") {
      // 未回答モード：kanji_progressに記録がない問題を取得する
      // ① まず回答済みの question_id を全件取得する
      const { data: progressData, error: progressError } = await supabase.from("kanji_progress").select("question_id");

      if (progressError) {
        console.error("進捗取得エラー:", progressError);
        return;
      }

      // ② 回答済みIDのリストを作る
      const answeredIds = progressData.map((p) => p.question_id);

      if (answeredIds.length === 0) {
        // 一度も回答していない場合は全件取得する
        result = await supabase.from("kanji_questions").select("*").order("id");
      } else {
        // 回答済みIDを除いた問題を取得する
        result = await supabase
          .from("kanji_questions")
          .select("*")
          .not("id", "in", `(${answeredIds.join(",")})`)
          .order("id");
      }
    } else if (mode === "review") {
      // 復習モード：next_review_atが今日以前の問題を取得する
      const today = new Date().toISOString().split("T")[0];

      // ① 今日復習すべき question_id を取得する
      const { data: progressData, error: progressError } = await supabase.from("kanji_progress").select("question_id").lte("next_review_at", today);

      if (progressError) {
        console.error("進捗取得エラー:", progressError);
        return;
      }

      const reviewIds = progressData.map((p) => p.question_id);

      if (reviewIds.length === 0) {
        // 復習すべき問題がない場合は空にする
        questions = [];
        loading = false;
        return;
      }

      // ② 該当する問題を取得する
      result = await supabase.from("kanji_questions").select("*").in("id", reviewIds).order("id");
    } else if (mode === "weak") {
      // 苦手一覧モード：incorrect_countが多い順に取得する
      // ① incorrect_countが1以上の progress を多い順に取得する
      const { data: progressData, error: progressError } = await supabase
        .from("kanji_progress")
        .select("question_id, incorrect_count")
        .gt("incorrect_count", 0)
        .order("incorrect_count", { ascending: false });

      if (progressError) {
        console.error("進捗取得エラー:", progressError);
        return;
      }

      if (progressData.length === 0) {
        questions = [];
        loading = false;
        return;
      }

      // ② 該当する問題を取得する
      const weakIds = progressData.map((p) => p.question_id);
      const { data: questionData, error: questionError } = await supabase.from("kanji_questions").select("*").in("id", weakIds);

      if (questionError) {
        console.error("問題取得エラー:", questionError);
        return;
      }

      // ③ incorrect_countの多い順に並び替える
      questions = weakIds.map((id) => questionData.find((q) => q.id === id)).filter((q) => q !== undefined);
      loading = false;
      return;
    }

    if (result.error) {
      console.error("取得エラー:", result.error);
      return;
    }

    questions = result.data;
    loading = false;
  }

  // {}で囲まれた部分を赤字のHTMLに変換する関数
  function toRedHtml(text) {
    // {と}で囲まれた部分を<span style="color:red">...</span>に変換する
    return text.replace(/\{(.+?)\}/g, '<span style="color: red;">$1</span>');
  }

  // 「答えを見る」ボタンを押したとき
  function handleShowAnswer() {
    showAnswer = true;
  }

  // 「書ける」ボタンを押したとき
  async function handleKnown() {
    // 結果を保存してから次へ進む
    await saveResult(currentQuestion.id, true);
    goNext();
  }

  // 「書けない」ボタンを押したとき
  async function handleUnknown() {
    // 結果を保存してから次へ進む
    await saveResult(currentQuestion.id, false);
    goNext();
  }

  // 正解回数から次回復習日を計算する関数
  function calcNextReviewAt(correctCount) {
    // 正解回数に応じて復習間隔（日数）を決める
    const intervals = [1, 3, 7, 14, 30];
    let days;

    if (correctCount <= intervals.length) {
      // 正解回数がテーブルの範囲内ならそのまま使う
      days = intervals[correctCount - 1];
    } else {
      // 6回目以降は前回の2倍（上限180日）
      days = Math.min(intervals[intervals.length - 1] * Math.pow(2, correctCount - intervals.length), 180);
    }

    // 今日の日付に日数を足して返す
    const date = new Date();
    date.setDate(date.getDate() + days);
    return date.toISOString().split("T")[0]; // YYYY-MM-DD形式
  }

  // 回答結果をSupabaseに保存する関数
  async function saveResult(questionId, isCorrect) {
    // ① kanji_results に回答履歴を追加する
    const { error: resultError } = await supabase.from("kanji_results").insert({ question_id: questionId, is_correct: isCorrect });

    if (resultError) {
      console.error("履歴の保存エラー:", resultError);
      return;
    }

    // ② kanji_progress の既存データを確認する
    const { data: existing, error: fetchError } = await supabase.from("kanji_progress").select("*").eq("question_id", questionId).maybeSingle(); // 0件のときエラーにしない

    if (fetchError) {
      console.error("進捗の取得エラー:", fetchError);
      return;
    }

    if (existing) {
      // ③ 既存データがある場合は更新する
      const newCorrectCount = isCorrect ? existing.correct_count + 1 : 0;
      const newIncorrectCount = isCorrect ? existing.incorrect_count : existing.incorrect_count + 1;
      const nextReviewAt = isCorrect ? calcNextReviewAt(newCorrectCount) : calcNextReviewAt(1);

      const { error: updateError } = await supabase
        .from("kanji_progress")
        .update({
          correct_count: newCorrectCount,
          incorrect_count: newIncorrectCount,
          next_review_at: nextReviewAt,
          updated_at: new Date().toISOString(),
        })
        .eq("question_id", questionId);

      if (updateError) {
        console.error("進捗の更新エラー:", updateError);
      }
    } else {
      // ④ 既存データがない場合は新規追加する
      const newCorrectCount = isCorrect ? 1 : 0;
      const newIncorrectCount = isCorrect ? 0 : 1;
      const nextReviewAt = isCorrect ? calcNextReviewAt(1) : calcNextReviewAt(1);

      const { error: insertError } = await supabase.from("kanji_progress").insert({
        question_id: questionId,
        correct_count: newCorrectCount,
        incorrect_count: newIncorrectCount,
        next_review_at: nextReviewAt,
      });

      if (insertError) {
        console.error("進捗の追加エラー:", insertError);
      }
    }
  }

  // 次のカードに進む
  function goNext() {
    if (mode === "weak") {
      // 苦手一覧モードはループする（最後のカードで最初に戻る）
      currentIndex = (currentIndex + 1) % questions.length;
      showAnswer = false;
      return;
    }

    if (currentIndex < questions.length - 1) {
      // まだ次のカードがある場合
      currentIndex++;
      showAnswer = false;
    } else {
      // 最後のカードだった場合
      currentIndex = questions.length; // 完了状態にする
    }
  }

  // 前のカードに戻る
  function goPrev() {
    if (currentIndex > 0) {
      currentIndex--;
      showAnswer = false;
    }
  }

  // ページ表示時にデータを取得する
  onMount(() => {
    fetchQuestions();
  });
</script>

{#if loading}
  <p style="text-align:center; margin-top: 80px;">読み込み中...</p>
{:else if questions.length === 0}
  <div class="empty">
    <p>該当する問題がありません</p>
    <a href="/">メニューに戻る</a>
  </div>
{:else if isCompleted}
  <!-- 完了画面 -->
  <div class="completed">
    <p>🎉 完了！</p>
    <button
      onclick={() => {
        currentIndex = 0;
        showAnswer = false;
      }}
    >
      もう一度
    </button>
  </div>
{:else}
  <!-- カード画面 -->
  <div class="card">
    <!-- 問題番号 -->
    <p class="index">{currentIndex + 1} / {questions.length}</p>

    <!-- 問題文（{}部分を赤字に変換して表示） -->
    <div class="question">
      {@html toRedHtml(currentQuestion.question)}
    </div>

    {#if showAnswer}
      <!-- 答えとタイ語（答えを見るを押した後） -->
      <div class="answer">
        <p>{@html toRedHtml(currentQuestion.answer)}</p>
        <p class="thai">{currentQuestion.thai}</p>
      </div>

      {#if mode !== "weak"}
        <!-- 苦手一覧モード以外は書ける・書けないボタンを表示する -->
        <div class="buttons">
          <button onclick={handleUnknown}>書けない</button>
          <button onclick={handleKnown}>書ける</button>
        </div>
      {/if}
    {:else}
      <!-- 答えを見るボタン -->
      <button onclick={handleShowAnswer}>答えを見る</button>
    {/if}

    <!-- 前へ・次へボタン -->
    <div class="nav">
      <button onclick={goPrev} disabled={currentIndex === 0}>← 前へ</button>
      <button onclick={goNext}>次へ →</button>
    </div>
  </div>
{/if}

<style>
  /* カード全体のレイアウト */
  .card {
    max-width: 480px;
    margin: 40px auto;
    padding: 24px;
    border: 1px solid #ccc;
    border-radius: 12px;
    text-align: center;
    font-family: sans-serif;
  }

  /* 問題番号 */
  .index {
    font-size: 14px;
    color: #999;
    margin-bottom: 16px;
  }

  /* 問題文 */
  .question {
    font-size: 28px;
    margin-bottom: 24px;
  }

  /* 答えエリア */
  .answer {
    background: #f9f9f9;
    border-radius: 8px;
    padding: 16px;
    margin-bottom: 16px;
  }

  /* 答えのテキスト */
  .answer p {
    font-size: 28px;
    margin: 0 0 8px;
  }

  /* タイ語 */
  .thai {
    font-size: 18px;
    color: #888;
  }

  /* 書ける・書けないボタンのエリア */
  .buttons {
    display: flex;
    gap: 12px;
    justify-content: center;
    margin-bottom: 16px;
  }

  /* 前へ・次へボタンのエリア */
  .nav {
    display: flex;
    gap: 12px;
    justify-content: center;
    margin-top: 24px;
  }

  /* ボタン共通スタイル */
  button {
    padding: 10px 24px;
    border: none;
    border-radius: 8px;
    font-size: 16px;
    cursor: pointer;
    background: #e0e0e0;
  }

  /* 書けるボタン */
  .buttons button:last-child {
    background: #4caf50;
    color: white;
  }

  /* 書けないボタン */
  .buttons button:first-child {
    background: #f44336;
    color: white;
  }

  /* 完了画面 */
  .completed {
    text-align: center;
    margin-top: 80px;
    font-size: 24px;
  }

  /* 問題が0件のとき */
  .empty {
    text-align: center;
    margin-top: 80px;
    font-size: 18px;
  }
</style>
