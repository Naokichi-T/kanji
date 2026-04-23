<script>
  import favicon from "$lib/assets/favicon.svg";
  import { page } from "$app/stores";
  let { children } = $props();
</script>

<svelte:head>
  <link rel="icon" href={favicon} />
  <title>漢字練習</title>
</svelte:head>

<!-- メニューに戻るリンク（メニュー画面では表示しない） -->
{#if $page.url.pathname !== "/"}
  <a href="/" class="back-link">← メニューに戻る</a>
{/if}

{@render children()}

<style>
  /* メニューに戻るリンク */
  .back-link {
    display: inline-block;
    margin: 16px 24px;
    font-size: 14px;
    color: #888;
    text-decoration: none;
  }

  .back-link:hover {
    color: #333;
  }
</style>
