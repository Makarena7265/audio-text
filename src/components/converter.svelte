<script lang="ts">
  import { FFmpeg } from "@ffmpeg/ffmpeg";
  import { fetchFile, toBlobURL } from "@ffmpeg/util";
  import { onMount } from "svelte";

  let outputFile: string | null;
  let isConverting = false;
  let ffmpeg: FFmpeg;
  let loaded = false;
  let tg: Telegram["WebApp"];

  const initFFmpeg = async () => {
    ffmpeg = new FFmpeg();
    const baseURL = "https://unpkg.com/@ffmpeg/core@0.12.6/dist/esm";
    ffmpeg.on("log", ({ message }) => {
      console.log(message);
    });
    await ffmpeg.load({
      coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, "text/javascript"),
      wasmURL: await toBlobURL(
        `${baseURL}/ffmpeg-core.wasm`,
        "application/wasm",
      ),
    });
  };

  const convertToOgg = async (file: File) => {
    // Загружаем файл в FFmpeg
    const fileName = file.name;
    const outputName = fileName.split(".").slice(0, -1).join(".") + ".ogg";
    ffmpeg.writeFile(fileName, await fetchFile(file));

    await ffmpeg.exec(["-i", fileName, outputName]);
    const data = await ffmpeg.readFile(outputName);
    const audio = new File([data], outputName, { type: "audio/ogg" });

    outputFile = URL.createObjectURL(audio);
    return audio;
  };

  async function handleSubmit(e: SubmitEvent) {
    const form = e.target as HTMLFormElement;
    const formData = new FormData(form);
    const file = formData.get("file") as File;
    isConverting = true;
    const audio = await convertToOgg(file);
    const base64 = await convertToBase64Async(audio);
    tg.sendData(base64);
    isConverting = false;
  }

  async function convertToBase64Async(file: File): Promise<string> {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = () => {
        if (reader.result === null) return reject("Failed to read file");
        if (reader.result instanceof ArrayBuffer)
          return resolve(reader.result.toString());
        resolve(reader.result);
      };
      reader.onerror = (error) => reject(error);
      reader.readAsDataURL(file);
    });
  }

  onMount(async () => {
    await initFFmpeg();
    tg = Telegram.WebApp;
    loaded = true;
  });
</script>

<h1>Конвертация аудио в формат OGG</h1>

{#if loaded}
  <form on:submit|preventDefault={handleSubmit}>
    <input type="file" name="file" accept="audio/*" />
    <button disabled={isConverting}>
      {isConverting ? "Конвертация..." : "Конвертировать"}
    </button>
  </form>
{/if}

{#if outputFile}
  <div class="output">
    <h3>Результат:</h3>
    <audio controls>
      <source src={outputFile} type="audio/ogg" />
      Ваш браузер не поддерживает воспроизведение OGG.
    </audio>
  </div>
{/if}

<style>
  .output {
    margin-top: 1rem;
  }
</style>
