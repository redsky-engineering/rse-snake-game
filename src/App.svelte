<script>
  import SnakeGame from './lib/SnakeGame.svelte'
</script>

<main>
  <header>
    <img src="/logo.svg" alt="RedSky" class="logo" />
    <h1>Snake</h1>
  </header>
  <SnakeGame />
</main>

<style>
  main {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 0.6rem;
    width: 100%;
    height: 100vh;
    padding: 0.75rem;
  }

  header {
    display: flex;
    align-items: center;
    gap: 0.6rem;
    flex-shrink: 0;
  }

  .logo {
    height: 1.75rem;
    width: auto;
    opacity: 0.85;
  }

  h1 {
    font-size: 1.75rem;
    letter-spacing: 0.14em;
    color: #cc1414;
    text-transform: uppercase;
    font-weight: 800;
  }
</style>
