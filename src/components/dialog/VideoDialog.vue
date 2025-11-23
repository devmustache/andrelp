<template>
  <Transition name="fade">
    <div
      v-show="$show.show && $show.id === video_id"
      class="bg-dark-blur z-1000 pointer-events-auto fixed inset-0 grid w-full cursor-pointer place-items-center"
      @click="pauseVideo()"
    >
      <div @click.stop class="container-md relative">
        <div class="overflow-hidden rounded shadow-xl relative">

          <!-- Spinner loading só no mobile -->
          <div v-if="loading && isMobile" class="absolute inset-0 z-20 flex items-center justify-center bg-black/30">
            <div class="spinner"></div>
          </div>

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
import { ref, watch } from "vue";
import "plyr/dist/plyr.css";
import { useStore } from "@nanostores/vue";
import { showVideo } from "@src/store";
import { disableBodyScroll, enableBodyScroll } from "body-scroll-lock";

const $show = useStore(showVideo);

const props = defineProps({
  video_id: { type: String },
  embed: { type: String },
});

const container = ref(null);
const videoPlayer = ref(null);
const loading = ref(false);
let Plyr;

// Detecta mobile
const isMobile = /iPhone|iPad|iPod|Android/i.test(navigator.userAgent);

// Função para abrir modal e tocar vídeo (chamada pelo clique do usuário)
export const openVideo = async () => {
  showVideo.set({ id: props.video_id, show: true });
  disableBodyScroll(document.body);
  await playVideo();
};

const pauseVideo = () => {
  showVideo.set({ id: $show.value.id, show: false });

  if (isMobile && videoPlayer.value) {
    videoPlayer.value.destroy();
    videoPlayer.value = null;
    container.value.innerHTML = ""; // limpa iframe
    loading.value = false;
  } else if (videoPlayer.value) {
    videoPlayer.value.pause();
  }

  enableBodyScroll(document.body);
};

const playVideo = async () => {
  if (!Plyr) Plyr = (await import("plyr")).default;

  if (!videoPlayer.value) {
    if (isMobile) loading.value = true;

    videoPlayer.value = new Plyr(container.value, {
      playsinline: isMobile ? true : 0,
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

    if (isMobile) {
      // Garante autoplay no mobile dentro da interação do usuário
      videoPlayer.value.on("canplay", () => {
        videoPlayer.value.play().catch(() => {});
        loading.value = false;
      });
    } else {
      // Desktop continua usando ready
      videoPlayer.value.on("ready", () => {
        videoPlayer.value.play();
      });
    }
  } else {
    videoPlayer.value.play();
  }
};

// Watcher para fechar modal via store
watch(
  $show,
  (val) => {
    if (!val.show && val.id === props.video_id) {
      enableBodyScroll(document.body);
    }
  },
  { immediate: true }
);
</script>

<style>
.z-1000 {
  z-index: 100;
}

/* Spinner animado */
@keyframes spin {
  to { transform: rotate(360deg); }
}
.spinner {
  border: 4px solid white;
  border-top-color: transparent;
  border-radius: 50%;
  width: 3rem;
  height: 3rem;
  animation: spin 1s linear infinite;
}
</style>
