<script>
  import SnakeGame from './lib/SnakeGame.svelte'
</script>

<main>
  <h1>Snake</h1>
  <SnakeGame />
</main>

<style>
  main {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1rem;
  }

  h1 {
    font-size: 2rem;
    letter-spacing: 0.1em;
    color: #4ade80;
  }
</style>
