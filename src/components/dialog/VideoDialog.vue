<template>
  <Transition name="fade">
    <div
      v-show="$show.show && $show.id === video_id"
      class="bg-dark-blur z-1000 pointer-events-auto fixed inset-0 grid w-full cursor-pointer place-items-center"
      @click="pauseVideo"
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
          @click="pauseVideo"
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
  video_id: { type: String },
  embed: { type: String },
  className: { type: String },
});

const container = ref(null);
const videoPlayer = ref(null);
const PlyrModule = ref(null);

// Carrega Plyr na montagem
onMounted(async () => {
  try {
    const plyr = await import("plyr");
    PlyrModule.value = plyr.default;
  } catch (error) {
    console.error("Falha ao carregar o Plyr:", error);
  }
});

/**
 * Adicionada: Função simples para verificar se o dispositivo é mobile.
 */
const isMobile = () => {
  if (typeof window === 'undefined') return false; // Segurança para SSR
  return /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(
    navigator.userAgent
  );
};


const pauseVideo = () => {
  if (videoPlayer.value) {
    videoPlayer.value.pause();
    // Opcional: destrói apenas no mobile ou se quiser reset completo
    // videoPlayer.value.destroy(); videoPlayer.value = null;
  }
  showVideo.set({ id: $show.value.id, show: false });
};

const playVideo = async () => {
  if (!PlyrModule.value) {
    try {
      const plyr = await import("plyr");
      PlyrModule.value = plyr.default;
    } catch (error) {
      console.error("Falha ao recarregar o Plyr:", error);
      return;
    }
  }

  if (!container.value) return;

  if (!videoPlayer.value) {
    videoPlayer.value = new PlyrModule.value(container.value, {
      playsinline: 1, // mobile inline
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
        iv_load_policy: 3,
        modestbranding: 1,
        showinfo: 0,
        rel: 0,
        enablejsapi: 1,
        noCookie: true,
      },
    });

    videoPlayer.value.on("ready", () => {
      // ✅ MODIFICAÇÃO: Tenta dar play APENAS se NÃO for mobile.
      if (!isMobile()) {
        videoPlayer.value.play().catch(() => {});
      }
    });
  } else {
    // ✅ MODIFICAÇÃO: Tenta dar play APENAS se NÃO for mobile.
    if (!isMobile()) {
      videoPlayer.value.play().catch(() => {});
    }
  }
};

// Watcher para abrir/fechar modal
watch(
  $show,
  (val) => {
    if (val.show && val.id === props.video_id) {
      playVideo();
      disableBodyScroll(document.body);
    } else if (!val.show && val.id === props.video_id) {
      enableBodyScroll(document.body);
    }
  },
  { immediate: true }
);
</script>

<style>
.z-1000 { z-index: 100; }
.bg-dark-blur { background: rgba(0, 0, 0, 0.7); backdrop-filter: blur(6px); }
</style>