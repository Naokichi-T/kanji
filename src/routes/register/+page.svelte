<script>
  import { supabase } from "$lib/supabase.js";
  import { onMount } from "svelte";

  // 入力フォームの値
  let inputQuestion = $state("");
  let inputAnswer = $state("");
  let inputThai = $state("");

  // 常用漢字チェック用の入力
  let inputKanji = $state("");
  let kanjiCheckResult = $state("");

  // 直近10件の登録リスト
  let recentQuestions = $state([]);

  // 登録中かどうか
  let saving = $state(false);

  // プレビュー用（{}を赤字HTMLに変換する）
  function toRedHtml(text) {
    // {と}で囲まれた部分を<span style="color:red">...</span>に変換する
    return text.replace(/\{(.+?)\}/g, '<span style="color: red;">$1</span>');
  }

  // 常用漢字かどうかを判定する関数
  function isJoyoKanji(char) {
    // 常用漢字2136字のUnicodeコードポイント範囲で判定する
    const code = char.codePointAt(0);
    // CJK統合漢字の範囲（大まかな判定）
    return code >= 0x4e00 && code <= 0x9fff;
  }

  // 常用漢字チェックボタンを押したとき
  function handleKanjiCheck() {
    const char = inputKanji.trim();

    if (char.length === 0) {
      kanjiCheckResult = "漢字を入力してください";
      return;
    }
    if (char.length > 1) {
      kanjiCheckResult = "1字だけ入力してください";
      return;
    }
    if (isJoyoKanji(char)) {
      kanjiCheckResult = `「${char}」は常用漢字の範囲内です ✅`;
    } else {
      kanjiCheckResult = `「${char}」は常用漢字の範囲外です ❌`;
    }
  }

  // 直近10件を取得する関数
  async function fetchRecentQuestions() {
    const { data, error } = await supabase.from("kanji_questions").select("*").order("created_at", { ascending: false }).limit(10);

    if (error) {
      console.error("取得エラー:", error);
      return;
    }

    recentQuestions = data;
  }

  // 登録ボタンを押したとき
  async function handleRegister() {
    // 問題と答えは必須
    if (inputQuestion.trim() === "" || inputAnswer.trim() === "") {
      alert("問題と答えは必須です");
      return;
    }

    saving = true;

    const { error } = await supabase.from("kanji_questions").insert({
      question: inputQuestion.trim(),
      answer: inputAnswer.trim(),
      thai: inputThai.trim(),
    });

    if (error) {
      console.error("登録エラー:", error);
      alert("登録に失敗しました");
      saving = false;
      return;
    }

    // 登録成功したら入力をクリアしてリストを更新する
    inputQuestion = "";
    inputAnswer = "";
    inputThai = "";
    saving = false;
    await fetchRecentQuestions();
  }

  // 削除ボタンを押したとき
  async function handleDelete(id) {
    const ok = confirm("この問題を削除しますか？");
    if (!ok) return;

    const { error } = await supabase.from("kanji_questions").delete().eq("id", id);

    if (error) {
      console.error("削除エラー:", error);
      alert("削除に失敗しました");
      return;
    }

    // 削除後にリストを更新する
    await fetchRecentQuestions();
  }

  // ページ表示時に直近10件を取得する
  onMount(() => {
    fetchRecentQuestions();
  });
</script>

<!-- 常用漢字チェック -->
<div class="section">
  <h2>常用漢字チェック</h2>
  <div class="kanji-check">
    <input type="text" maxlength="1" placeholder="漢字1字" bind:value={inputKanji} />
    <button onclick={handleKanjiCheck}>チェックする</button>
    <button
      class="clear-btn"
      onclick={() => {
        inputKanji = "";
        kanjiCheckResult = "";
      }}>クリア</button
    >
  </div>
  {#if kanjiCheckResult}
    <p class="check-result">{kanjiCheckResult}</p>
  {/if}
  <a href="https://kanji.jitenon.jp/cat/joyo" target="_blank" rel="noreferrer"> 常用漢字一覧を見る → </a>
</div>

<!-- 問題登録フォーム -->
<div class="section">
  <h2>問題を登録する</h2>
  <p class="hint">赤字にしたい部分を{"{}"}で囲んでください（例：お金を{"{かせ}"}ぐ）</p>

  <div class="form">
    <label>
      問題（ひらがな混じり）
      <input type="text" placeholder="例：お金を&#123;かせ&#125;ぐ" bind:value={inputQuestion} />
    </label>

    <!-- プレビュー -->
    {#if inputQuestion}
      <p class="preview">プレビュー：{@html toRedHtml(inputQuestion)}</p>
    {/if}

    <label>
      答え（漢字混じり）
      <input type="text" placeholder="例：お金を&#123;稼&#125;ぐ" bind:value={inputAnswer} />
    </label>

    <!-- プレビュー -->
    {#if inputAnswer}
      <p class="preview">プレビュー：{@html toRedHtml(inputAnswer)}</p>
    {/if}

    <label>
      タイ語（参考情報）
      <input type="text" placeholder="例：หาเงิน" bind:value={inputThai} />
    </label>

    <button onclick={handleRegister} disabled={saving}>
      {saving ? "登録中..." : "登録する"}
    </button>
  </div>
</div>

<!-- 直近10件のリスト -->
<div class="section">
  <h2>直近の登録（最大10件）</h2>
  {#if recentQuestions.length === 0}
    <p>まだ登録がありません</p>
  {:else}
    <ul class="recent-list">
      {#each recentQuestions as q}
        <li>
          <span>
            {@html toRedHtml(q.question)} → {@html toRedHtml(q.answer)}
            {#if q.thai}
              <span class="thai-label">（{q.thai}）</span>
            {/if}
          </span>
          <button class="delete-btn" onclick={() => handleDelete(q.id)}>削除</button>
        </li>
      {/each}
    </ul>
  {/if}
</div>

<style>
  /* 全体のレイアウト */
  :global(body) {
    font-family: sans-serif;
    max-width: 600px;
    margin: 0 auto;
    padding: 24px;
  }

  /* セクションごとの余白 */
  .section {
    margin-bottom: 40px;
  }

  h2 {
    font-size: 18px;
    margin-bottom: 12px;
  }

  /* 常用漢字チェックエリア */
  .kanji-check {
    display: flex;
    gap: 8px;
    margin-bottom: 8px;
  }

  .kanji-check input {
    width: 60px;
    font-size: 20px;
    text-align: center;
    padding: 4px;
  }

  /* チェック結果 */
  .check-result {
    font-size: 16px;
    margin: 8px 0;
  }

  /* ヒントテキスト */
  .hint {
    font-size: 13px;
    color: #888;
    margin-bottom: 12px;
  }

  /* フォームのレイアウト */
  .form {
    display: flex;
    flex-direction: column;
    gap: 16px;
  }

  label {
    display: flex;
    flex-direction: column;
    gap: 4px;
    font-size: 14px;
  }

  input[type="text"] {
    padding: 8px;
    font-size: 16px;
    border: 1px solid #ccc;
    border-radius: 6px;
  }

  /* プレビュー */
  .preview {
    font-size: 18px;
    color: #333;
    margin: 0;
    padding: 8px;
    background: #f9f9f9;
    border-radius: 6px;
  }

  button {
    padding: 10px 24px;
    font-size: 16px;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    background: #4caf50;
    color: white;
  }

  button:disabled {
    background: #aaa;
    cursor: not-allowed;
  }

  /* 直近リスト */
  .recent-list {
    list-style: none;
    padding: 0;
    margin: 0;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .recent-list li {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 8px 12px;
    background: #f9f9f9;
    border-radius: 6px;
    font-size: 16px;
  }

  /* 削除ボタン */
  .delete-btn {
    padding: 4px 12px;
    font-size: 13px;
    background: #f44336;
    color: white;
    border-radius: 6px;
    flex-shrink: 0;
  }

  /* クリアボタン */
  .clear-btn {
    background: #e0e0e0;
    color: #333;
  }

  /* 直近リストのタイ語 */
  .thai-label {
    font-size: 18px;
    color: #888;
    margin-left: 4px;
  }
</style>
