<template>
  <div><div :id="elementId" /></div>
</template>

<script>
import player from "youtube-player";
import PlayerMixin from "./PlayerMixin";

const UNSTARTED = -1;
const ENDED = 0;
const PLAYING = 1;
const PAUSED = 2;
const BUFFERING = 3;
const CUED = 5;

let pid = 1;
export default {
    name: "YoutubePlayer",
    mixins: [PlayerMixin],
    props: {
        videoId: String,
        playerVars: {
            type: Object,
            default: () => ({}),
        },
    },
    data() {
        pid += 1;
        return {
            events: {
                [UNSTARTED]: "unstarted",
                [PLAYING]: "playing",
                [PAUSED]: "paused",
                [ENDED]: "ended",
                [BUFFERING]: "buffering",
                [CUED]: "cued",
            },
            elementId: `youtube-player-${pid}`,
        };
    },
    computed: {
    },
    watch: {
        videoId: "updatePlayer",
        mute: "setMute",
    },
    beforeDestroy() {
        if (this.player !== null && this.player.destroy) {
            this.player.destroy();
            delete this.player;
        }
    },
    async mounted() {
        window.YTConfig = {
            host: "https://www.youtube.com/iframe_api",
        };

        const host = "https://www.youtube.com";

        this.player = player(this.elementId, {
            host,
            width: this.width,
            height: this.height,
            videoId: this.videoId,
            playerVars: this.playerVars,
            origin: window.origin,
        });

        this.player.on("ready", (e) => this.playerReady(e.target));
        this.player.on("stateChange", this.playerStateChange);
        this.player.on("error", this.playerError);
    },
    methods: {
        playerReady(player) {
            // Call parent mixin's playerReady
            PlayerMixin.methods.playerReady.call(this, player);

            // Set quality to HD after player is ready
            setTimeout(() => {
                this.setQualityToHD();
            }, 1000);
        },
        playerStateChange(e) {
            if (e.data !== null && e.data !== UNSTARTED) {
                this.$emit(this.events[e.data], e.target);

                // Try to set quality when video starts playing
                if (e.data === PLAYING) {
                    this.setQualityToHD();
                }
            }
        },
        async setQualityToHD() {
            try {
                // Try to set quality to HD using available methods
                const availableQualities = await this.player.getAvailableQualityLevels();
                console.log("Available qualities:", availableQualities);

                // Prefer higher qualities
                const preferredQualities = ['hd1080', 'hd720', 'large'];
                const qualityToSet = preferredQualities.find(q => availableQualities.includes(q)) || availableQualities[0];

                if (qualityToSet && typeof this.player.setPlaybackQualityRange === 'function') {
                    console.log("Setting quality range to:", qualityToSet);
                    this.player.setPlaybackQualityRange(qualityToSet);
                }
                if (qualityToSet && typeof this.player.setPlaybackQuality === 'function') {
                    console.log("Setting quality to:", qualityToSet);
                    this.player.setPlaybackQuality(qualityToSet);
                }
            } catch (error) {
                console.warn("Could not set video quality:", error);
            }
        },
        updatePlayer(videoId) {
            if (!videoId) {
                this.player.stopVideo();
                return;
            }

            const params = { videoId };

            if (typeof this.playerVars.start === "number") {
                params.startSeconds = this.playerVars.start;
            }

            if (typeof this.playerVars.end === "number") {
                params.endSeconds = this.playerVars.end;
            }

            console.log("LoadVideoByID in Vue-Youtube", params);

            if (this.playerVars.autoplay === 1) {
                this.player.loadVideoById(params);
                return;
            }

            this.player.cueVideoById(params);
        },
        setMute(value) {
            if (!this.player) return;
            value ? this.player.mute() : this.player.unMute();
        },
        getCurrentTime() {
            return this.player?.getCurrentTime();
        },
        getPlaybackRate() {
            return this.player.getPlaybackRate();
        },
        getVolume() {
            return this.player.getVolume();
        },
        isMuted() {
            return this.player.isMuted();
        },
        seekTo(t) {
            this.player.seekTo(t);
        },
        setPlaying(playing) {
            if (!this.player) return;
            !playing ? this.player.pauseVideo() : this.player.playVideo();
        },
        async sendLikeEvent() {
            const iframe = await this.player.getIframe();
            iframe.contentWindow.postMessage({ event: "likeVideo" }, "*");
        },
    },
};
</script>
