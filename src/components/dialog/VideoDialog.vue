<template>
  <Transition name="fade">
    <div
      v-show="$show.show && $show.id === video_id"
      class="bg-dark-blur z-1000 pointer-events-auto fixed inset-0 grid w-full cursor-pointer place-items-center"
      @click="pauseVideo()"
    >
      <div @click.stop class="container-md relative">
        <div class="overflow-hidden rounded shadow-xl">
          <div
            class="w-full"
            ref="container"
            :data-plyr-provider="embed"
            :data-plyr-embed-id="video_id"
          ></div>
        </div>

        <button
          class="btn btn-icon surface-dark btn-absolute -right-1 -top-1 z-10 grid h-10 w-10 place-items-center"
          @click="pauseVideo()"
        >
          <slot />
        </button>
      </div>
    </div>
  </Transition>
</template>

<script setup>
import { ref, watch, onMounted } from "vue";

import "plyr/dist/plyr.css";

import { useStore } from "@nanostores/vue";
import { showVideo } from "@src/store";

import { disableBodyScroll, enableBodyScroll } from "body-scroll-lock";

const $show = useStore(showVideo);

const props = defineProps({
  video_id: {
    type: String,
  },
  embed: {
    type: String,
  },
  className: {
    type: String,
  },
});

const container = ref(null);
const videoPlayer = ref(null);
const loading = ref(false);
let Plyr;

onMounted(async () => {});

const pauseVideo = () => {
  showVideo.set({
    id: $show.value.id,
    show: false,
  });
  if (videoPlayer.value) videoPlayer.value.pause();
};

const playVideo = async () => {
  /* allowCookie.value = checkCookie(); */
  if (!Plyr) Plyr = (await import("plyr")).default;

  if (!videoPlayer.value) {
    // 1. Criação do player (ocorre na primeira vez)
    loading.value = true;
    videoPlayer.value = new Plyr(container.value, {
      playsinline: 0,
      settings: ["loop"],
      iconUrl: "/icons/plyr.svg",
      controls: [
        "play",
        "progress",
        "current-time",
        "mute",
        "volume",
        "airplay",
        "fullscreen",
      ],
      youtube: {
        origin: window.location.host,
        iv_load_policy: 3,
        modestbranding: 1,
        showinfo: 0,
        rel: 0,
        enablejsapi: 1,
        noCookie: true,
      },
    });

    // 2. Tenta dar o play APENAS quando o Plyr estiver pronto.
    // O navegador móvel pode bloquear este autoplay, mas o player estará inicializado.
    videoPlayer.value.on("ready", function (event) {
      videoPlayer.value.play().catch(error => {
         console.warn("Autoplay bloqueado pelo navegador na primeira tentativa.", error);
         loading.value = false;
      });
      // A flag de loading será setada para false aqui ou em 'loadeddata'
    });

    // Adiciona listener para garantir que o loading é desativado quando dados são carregados
    videoPlayer.value.on("loadeddata", function () {
      loading.value = false;
    });

  } else {
    // 3. Tenta dar o play quando o player JÁ EXISTE (ocorre na segunda vez)
    // Isso tem maior chance de sucesso em móvel, pois o player já está injetado.
    videoPlayer.value.play().catch(error => {
      console.warn("Erro ao tentar tocar novamente. Bloqueio de mídia?", error);
    });
  }
};

watch(
  $show,
  (val) => {
    if (val.show && val.id === props.video_id) {
      playVideo();
      disableBodyScroll(document.body);
    }
    if (!val.show && val.id === props.video_id) {
      enableBodyScroll(document.body);
    }
  },
  { immediate: true },
);
</script>

<style>
.z-1000 {
  z-index: 100;
}
</style>