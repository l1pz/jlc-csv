<script>
  import { config } from "./config.js";

  let downloadProgressPercent = 0;

  // Source: https://javascript.info/fetch-progress
  async function downloadJlcCsv() {
    // Step 1: start the fetch and obtain a reader
    let response = await fetch(config.jlcCsvUrl);

    const reader = response.body.getReader();

    // Step 2: get total length
    const contentLength = +response.headers.get("Content-Length");

    // Step 3: read the data
    let receivedLength = 0; // received that many bytes at the moment
    let chunks = []; // array of received binary chunks (comprises the body)
    while (true) {
      const { done, value } = await reader.read();

      if (done) break;

      chunks.push(value);
      receivedLength += value.length;

      downloadProgressPercent = (receivedLength / contentLength) * 100;
      console.log(downloadProgressPercent);
    }

    // Step 4: concatenate chunks into single Uint8Array
    let chunksAll = new Uint8Array(receivedLength); // (4.1)
    let position = 0;
    for (let chunk of chunks) {
      chunksAll.set(chunk, position); // (4.2)
      position += chunk.length;
    }

    // Step 5: decode into a string
    let result = new TextDecoder("utf-8").decode(chunksAll);

    return result;
  }

  let files;
  async function loadBomCsv() {
    let text = await files[0].text();
    return text;
  }

  let quantity = 1;
  $: quantity = typeof quantity == "number" && quantity < 1 ? 1 : quantity;
</script>

<svelte:head>
  <link
    rel="stylesheet"
    href="https://cdn.jsdelivr.net/npm/bulma@0.9.3/css/bulma.min.css"
  />
  <script
    src="https://kit.fontawesome.com/7715a8d6b4.js"
    crossorigin="anonymous"></script>
</svelte:head>

<div class="container">
  <h1 class="title is-2 mx-2 mt-4">JLC Compontent Helper</h1>
  <!-- {#await downloadJlcCsv()}
    <div class="mx-2">
      <p>
        Downloading JLC Compontent info -
        <span class="has-text-weight-bold">
          {Math.round(downloadProgressPercent)}%
        </span>
      </p>
      <progress
        class="progress is-primary"
        value={downloadProgressPercent}
        max="100">{Math.round(downloadProgressPercent)}</progress
      >
    </div>
  {/await} -->

  <div class="mx-2">
    <p>Downloading finished. Please upload BOM.csv.</p>

    <div class="file">
      <label class="file-label">
        <input
          class="file-input"
          type="file"
          name="resume"
          accept=".csv"
          bind:files
        />
        <span class="file-cta">
          <span class="file-icon">
            <i class="fas fa-upload" />
          </span>
          <span class="file-label">Upload BOM.csv</span>
        </span>
      </label>
    </div>

    <div class="field mt-4">
      <label class="label">Quantity</label>
      <div class="control">
        <input class="input" type="number" bind:value={quantity} />
      </div>
    </div>

    {#if files && files[0]}
      {#await loadBomCsv()}
        <p>Parsing BOM.csv...</p>
      {:then}
        {"asd"}
      {:catch error}
        <p class="has-text-danger">{error.message}</p>
      {/await}
    {/if}
  </div>
</div>
