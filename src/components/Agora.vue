<template>
  <div class="flex">
    <div class="flex-1 space-y-3 px-20 py-10">
      <div class="field">
        <label class="label">App ID</label>
        <div class="control">
          <input class="input" type="text" v-model="appId" />
        </div>
      </div>

      <div class="field">
        <label class="label">Channel</label>
        <div class="control">
          <input class="input" type="text" v-model="channel" />
        </div>
      </div>

      <div class="field">
        <label class="label">Token</label>
        <div class="control">
          <input class="input" type="text" v-model="token" />
        </div>
      </div>

      <div class="space-x-3">
        <button
          class="button is-primary"
          @click="handleJoin"
          :disabled="joined || disableSubmit"
        >
          Join
        </button>
        <button
          class="button is-primary"
          @click="handleLeave"
          :disabled="!joined"
        >
          Leave
        </button>
        <button
          class="button is-primary"
          @click="handlePublish"
          :disabled="!joined"
        >
          Publish
        </button>
        <button
          class="button is-primary"
          @click="handleUnpublish"
          :disabled="!joined"
        >
          Unpublish
        </button>
      </div>
    </div>
    <div class="flex-1 space-y-3 px-20 py-10">
      <div class="field">
        <label class="label">UID</label>
        <div class="control">
          <input class="input" type="text" v-model="inputUid" />
        </div>
      </div>

      <div class="field">
        <label class="label">Camera</label>
        <div class="control">
          <div class="select">
            <select v-model="camId">
              <option
                v-for="cam in cams"
                :key="cam.deviceId"
                :value="cam.deviceId"
              >
                {{ cam.label }}</option
              >
            </select>
          </div>
        </div>
      </div>

      <div class="field">
        <label class="label">Microphone</label>
        <div class="control">
          <div class="select">
            <select v-model="micId">
              <option
                v-for="mic in mics"
                :key="mic.deviceId"
                :value="mic.deviceId"
              >
                {{ mic.label }}</option
              >
            </select>
          </div>
        </div>
      </div>

      <div class="flex">
        <div class="field flex-1">
          <label class="label">Codec</label>
          <div class="control">
            <label class="radio">
              <input type="radio" value="h264" v-model="codec" />
              h264
            </label>
            <label class="radio">
              <input type="radio" value="vp8" v-model="codec" />
              vp8
            </label>
          </div>
        </div>

        <div class="field  flex-1">
          <label class="label">Mode</label>
          <div class="control">
            <label class="radio">
              <input type="radio" value="live" v-model="mode" />
              live
            </label>
            <label class="radio">
              <input type="radio" value="rtc" v-model="mode" />
              rtc
            </label>
          </div>
        </div>
      </div>
    </div>
  </div>

  <div class="flex flex-wrap space-x-5 space-y-5">
    <div v-if="uid" id="local-player" class="player">
      <div>Your UID: {{ uid }}</div>
    </div>

    <div v-for="(user, uid) in remoteUsers" :key="uid" :id="uid" class="player">
      <div>UID: {{ uid }}</div>
    </div>
  </div>
</template>

<script lang="ts">
import { Vue } from "vue-class-component";
import AgoraRTC, {
  IAgoraRTCClient,
  IAgoraRTCRemoteUser,
  IMicrophoneAudioTrack,
  ILocalVideoTrack,
  SDK_CODEC,
  SDK_MODE
} from "agora-rtc-sdk-ng";

interface RemoteUser {
  [uid: string]: IAgoraRTCRemoteUser;
}

export default class Agora extends Vue {
  client!: IAgoraRTCClient;
  localAudioTrack!: IMicrophoneAudioTrack;
  localVideoTrack!: ILocalVideoTrack;
  remoteUsers: RemoteUser = {};
  uid = "";
  appId = "";
  token = "";
  channel = "";
  joined = false;
  inputUid = "";
  camId = "";
  cams: MediaDeviceInfo[] = [];
  micId = "";
  mics: MediaDeviceInfo[] = [];
  codec: SDK_CODEC = "h264";
  mode: SDK_MODE = "rtc";

  get disableSubmit() {
    return !this.appId || !this.token || !this.channel;
  }

  mounted() {
    this.getDevices();
  }

  async getDevices() {
    let devices: MediaDeviceInfo[] = [];
    try {
      devices = await AgoraRTC.getDevices();
    } catch (e) {
      console.error(e);
      return;
    }

    this.cams = [];
    this.mics = [];
    for (const device of devices) {
      if (device.kind === "videoinput") {
        this.cams.push(device);
      } else if (device.kind === "audioinput") {
        this.mics.push(device);
      }
    }
    if (this.cams.length) this.camId = this.cams[0].deviceId;
    if (this.mics.length) this.micId = this.mics[0].deviceId;
  }

  async handleJoin() {
    try {
      this.client = AgoraRTC.createClient({
        mode: this.mode,
        codec: this.codec
      });
    } catch (e) {
      console.error(e);
      return;
    }

    this.client.on("user-published", this.handleUserPublished);
    this.client.on("user-unpublished", this.handleUserUnpublished);

    try {
      this.uid = (
        await this.client.join(
          this.appId,
          this.channel,
          this.token || null,
          this.inputUid ? this.inputUid : null
        )
      ).toString();
      this.joined = true;
    } catch (e) {
      console.error(e);
      return;
    }

    try {
      this.localAudioTrack = await AgoraRTC.createMicrophoneAudioTrack({
        microphoneId: this.micId
      });
    } catch (e) {
      console.error(e);
    }

    try {
      this.localVideoTrack = await AgoraRTC.createCameraVideoTrack({
        cameraId: this.camId
      });
    } catch (e) {
      console.error(e);
    }

    if (this.localVideoTrack) {
      this.localVideoTrack.play("local-player");
      this.handlePublish();
    }
  }

  async handleLeave() {
    try {
      this.localAudioTrack.close();
      this.localVideoTrack.close();
    } catch (e) {
      console.error(e);
    }

    this.uid = "";

    this.remoteUsers = {};
    try {
      await this.client.leave();
      this.joined = false;
    } catch (e) {
      console.error(e);
    }
  }

  async handlePublish() {
    try {
      await this.client.publish([this.localAudioTrack, this.localVideoTrack]);
    } catch (e) {
      console.error(e);
    }
  }

  async handleUnpublish() {
    try {
      await this.client.unpublish([this.localAudioTrack, this.localVideoTrack]);
    } catch (e) {
      console.error(e);
    }
  }

  async handleUserPublished(user: IAgoraRTCRemoteUser, mediaType: "video") {
    try {
      await this.client.subscribe(user, mediaType);
    } catch (e) {
      console.error(e);
      return;
    }

    if (mediaType === "video") {
      this.handleRemoteVideo(user);
    } else if (mediaType === "audio") {
      this.handleRemoteAudio(user);
    }
  }

  async handleUserUnpublished(user: IAgoraRTCRemoteUser) {
    const id = user.uid;
    delete this.remoteUsers[id];
  }

  handleRemoteVideo(user: IAgoraRTCRemoteUser) {
    const remoteVideoTrack = user.videoTrack;
    if (!remoteVideoTrack) return;

    const uid = user.uid.toString();
    this.remoteUsers[uid] = user;
    this.$nextTick(() => {
      remoteVideoTrack.play(uid);
    });
  }

  handleRemoteAudio(user: IAgoraRTCRemoteUser) {
    const remoteAudioTrack = user.audioTrack;
    if (!remoteAudioTrack) return;

    remoteAudioTrack.play();
  }
}
</script>

<style scoped>
.player {
  width: 480px;
  height: 320px;
  margin: 20px;
}
</style>
