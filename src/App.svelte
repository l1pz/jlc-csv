<script>
  import { config } from "./config.js";
  import Papa from "papaparse";

  let downloadProgressPercent = 0;
  let jlcData;

  let selectedFiles;
  let bomCsv;

  let quantity = 1;
  $: quantity = typeof quantity == "number" && quantity < 1 ? 1 : quantity;

  let showTable = false;
  let generateTablePromise;
  let tableParts = [];
  let minMaxPcb = Infinity;

  // Source: https://javascript.info/fetch-progress
  async function downloadAndParseJlcCsv() {
    // Step 1: start the fetch and obtain a reader
    let response = await fetch(config.jlcCsvUrl);

    if (!response.ok)
      throw new Error("Couldn't download JLCPCB part information.");

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
    }

    // Step 4: concatenate chunks into single Uint8Array
    let chunksAll = new Uint8Array(receivedLength); // (4.1)
    let position = 0;
    for (let chunk of chunks) {
      chunksAll.set(chunk, position); // (4.2)
      position += chunk.length;
    }

    // Step 5: decode into a string
    const jlcCsv = new TextDecoder("utf-8").decode(chunksAll);

    // Step 6: parse csv string
    jlcData = Papa.parse(jlcCsv, { header: true }).data;
  }

  async function generateTable() {
    bomCsv = await selectedFiles[0].text();
    showTable = true;
    const bomData = Papa.parse(bomCsv, { header: true }).data;
    console.log(jlcData[0]);
    console.log(Object.keys(jlcData[0]));
    for (let spart of jlcData) {
      console.log(spart["LSCS Part"]);
    }
    tableParts = [];
    minMaxPcb = Infinity;
    for (let part of bomData) {
      const partId = part["Supplier Part"];
      if (partId === "") continue;

      const jlcPart = jlcData.find((x) => x["LCSC Part"] == partId);
      console.log(partId);
      if (!jlcPart) continue;

      const stock = jlcPart["Stock"];
      const quantity = part["Quantity"];
      const maxPcb = Math.floor(stock / quantity);
      if (minMaxPcb > maxPcb) minMaxPcb = maxPcb;
      part["Stock"] = stock;
      part["Max PCB"] = maxPcb;
      tableParts.push(part);
    }
  }

  function changeFile() {
    showTable = false;
    generateTablePromise = generateTable();
  }
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
  <h1 class="title is-2 mx-2 mt-4">JLCPCB Compontent Stock Helper</h1>
  {#await downloadAndParseJlcCsv()}
    <div class="mx-2">
      <p>
        Downloading and parsing JLCPCB compontent info -
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
  {:then}
    <div class="mx-2">
      <div class="columns">
        <div class="column">
          {#if !selectedFiles}
            <p>Downloading and parsing finished. Please upload a BOM file:</p>
          {:else}
            <p>Use a different BOM.csv:</p>
          {/if}
          <div class="file">
            <label class="file-label">
              <input
                class="file-input"
                type="file"
                name="resume"
                accept=".csv"
                bind:files={selectedFiles}
                on:change={changeFile}
              />
              <span class="file-cta">
                <span class="file-icon">
                  <i class="fas fa-upload" />
                </span>
                <span class="file-label">Upload BOM file</span>
              </span>
            </label>
            <span class="file-name">
              {selectedFiles && selectedFiles[0]
                ? selectedFiles[0].name
                : "BOM.csv"}
            </span>
          </div>
        </div>

        {#if selectedFiles && selectedFiles[0]}
          {#await generateTablePromise}
            <div class="column">
              <p>
                <i class="fas fa-cog fa-spin" />&nbsp;Processing CSV Files...
              </p>
            </div>
          {:then}
            {#if minMaxPcb < Infinity}
              <div class="column">
                <p>Max PCBs:</p>
                <div class="control">
                  <input
                    class="input"
                    type="number"
                    value={minMaxPcb}
                    readonly
                  />
                </div>
              </div>
            {/if}
            <div class="column">
              <p>Quantity:</p>
              <div class="field">
                <div class="control">
                  <input class="input" type="number" bind:value={quantity} />
                </div>
              </div>
            </div>
          {:catch error}
            <p class="has-text-danger">{error.message}</p>
          {/await}
        {/if}
      </div>
      {#if showTable}
        <table class="table">
          <thead>
            <tr>
              {#each Object.keys(tableParts[0]) as header}
                <th>{header}</th>
              {/each}
            </tr>
          </thead>
          <tbody>
            {#each tableParts as part}
              <tr
                class={part["Max PCB"] < quantity
                  ? "has-text-light has-background-danger-dark"
                  : ""}
              >
                {#each Object.keys(part) as key}
                  <td>{part[key]}</td>
                {/each}
              </tr>
            {/each}
          </tbody>
        </table>
      {/if}
    </div>
  {:catch error}
    <p class="has-text-danger">{error.message}</p>
  {/await}
</div>
