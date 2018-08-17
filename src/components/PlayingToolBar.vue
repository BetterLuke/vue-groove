<template>
  <div class="PlayingToolBar" :class="{'top':isShowList,'bottom':!isShowList,
          'moveTop':moveShowList===true,'moveBottom':moveShowList===false}" ref="PlayingToolBar">
    <div class="file-info" ref="fileInfo">
      <div class="mask"></div>
      <div class="cover">
        <img :src="playingFile.imgUrl" alt="">
      </div>
      <div class="info">
        <h5>
          {{playingFile.title}}
        </h5>
        <p>{{playingFile.p}}</p>
      </div>
    </div>

    <div ref="operations" class="operations">
      <div class="progress">
        <p class="current">{{currentTimeStr}}</p>
        <div class="slider">
          <Slider :value="timePercent" @on-change="handleSelectTime" :tip-format="formatProgress"></Slider>
        </div>
        <p class="total">{{durationTimeStr}}</p>
      </div>

      <div class="buttons">
        <div class="left">
          <!--<div class="operate random">
            <Icon type="ios-shuffle"/>
          </div>-->
          <Random class="operate"></Random>
          <div class="operate prev" @click="toPrev">
            <Icon type="ios-skip-backward-outline"/>
          </div>
          <div class="operate play" @click="togglePlay">
            <!--<Icon type="ios-play-outline"/>-->
            <Icon v-if="!playing" type="ios-play-outline"/>
            <Icon v-if="playing" type="ios-pause-outline"/>
          </div>
          <div class="operate next" @click="toNext">
            <Icon type="ios-skip-forward-outline"/>
          </div>
          <Loop class="operate loop"></Loop>
          <!--<div class="operate repeat">
            <Icon type="ios-repeat"/>
          </div>-->

          <div class="operate volume" @click="showVolumeModal">
            <transition name="volume-modal">
              <div class="volume-modal" v-if="volumeModalShow" ref="volumeModalShow">
                <div class="opts">
                  <div class="icon">
                    <!--<Icon type="ios-volume-down"/>-->
                    <VolumeIcon :click-to-mute="true"></VolumeIcon>
                  </div>
                  <div class="slide">
                    <Slider :value="volume" @on-input="handleChangeVolume"></Slider>
                  </div>
                </div>
              </div>
            </transition>
            <VolumeIcon></VolumeIcon>
          </div>

          <div class="operate more" @click="showOperateMenu">
            <Icon type="ios-more"/>
          </div>

        </div>
        <div class="right">
          <div class="operate play-list" @click="toggleList">
            <!--<div class="operate play-list" @click="$emit('toggleList')">-->
            <Icon type="ios-list"/>
          </div>
          <div class="operate full-screen" @click="toggleFullScreen">
            <Icon type="ios-resize"/>
          </div>
        </div>
      </div>
    </div>


    <div class="arrow" :class="{'up':$route.params['light']==='light'}" @click="$emit('toggleList')">
      <!--<Icon type="ios-arrow-down"/>-->
      <img src="/res/vcplayer/arrow.svg" alt="">
    </div>
  </div>
</template>

<script lang="ts">
  import {Component, Prop, Vue} from "vue-property-decorator";
  import {State, Mutation, Action, namespace} from "vuex-class";
  import {getTimeStr} from "../store/modules/audio";
  import {dropDownMenu, editPlayListModal, isInSelf, toggleFullScreen} from "../utils/utils";
  import VolumeIcon from "./VolumeIcon.vue";
  import Loop from "./Operation/Loop.vue";
  import Random from "./Operation/Random.vue";
  import {File} from "../store/modules/file";

  const audioModule = namespace("audio");
  const homeModule = namespace("home");
  const playListModule = namespace("playList");

  @Component({
    components: {Random, Loop, VolumeIcon},
    /*    props: {
          isShowList: Boolean,
          moveShowList: Boolean
        }*/
  })
  export default class PlayingToolBar extends Vue {
    @Prop(Boolean) isShowList !: boolean;
    @Prop(Boolean) moveShowList !: boolean;

    @audioModule.State playing!: boolean;
    @audioModule.State currentTime!: number;
    @audioModule.State duration!: number;
    @audioModule.State volume!: number;
    @playListModule.State playingList!: Array<File>;
    @audioModule.Getter currentTimeStr!: string;
    @audioModule.Getter durationTimeStr!: string;
    @audioModule.Getter timePercent!: number;
    @audioModule.Mutation handleSelectTime!: any;
    @audioModule.Mutation handleChangeVolume!: any;
    @homeModule.State playingFile !: any;
    @homeModule.State isDark!: boolean;

    @audioModule.Action toPrev!: any;
    @audioModule.Action toNext!: any;

    volumeModalShow: boolean = false;

    // menuX = 0
    // menuY = 0

    toggleFullScreen: Function = toggleFullScreen;

    public mounted() {
      if (!this.isDark) {
        window.addEventListener("mousemove", this.toMinListener);
      }
    }

    toggleList() {

      if (this.isDark) {
        window.addEventListener("mousemove", this.toMinListener);

      } else {
        window.removeEventListener("mousemove", this.reduce);
        window.removeEventListener("mousemove", this.toMinListener);
      }

      this.$emit("toggleList");
    }

    toMinListener() {
      clearTimeout(this.timer);
      this.timer = window.setTimeout(this.toMini, 3000);
    }

    toMini() {
      if (this.isDark) return;
      const fileInfo = <HTMLElement>this.$refs.fileInfo;
      const operations = <HTMLElement>this.$refs.operations;
      const header =<HTMLElement> document.querySelector('.Playing>.wrap>.header')
      header.style.opacity = '0'
      if (!fileInfo||!operations) return;
      const operateHeight = operations.offsetHeight;
      fileInfo.style.transform = `translateY(${operateHeight}px)`;
      operations.style.visibility = "hidden";

      window.removeEventListener("mousemove", this.reduce);
      window.addEventListener("mousemove", this.reduce);
    }

    reduce() {

      const fileInfo = <HTMLElement>this.$refs.fileInfo;
      const operations = <HTMLElement>this.$refs.operations;
      const header =<HTMLElement> document.querySelector('.Playing>.wrap>.header')
      header.style.opacity = '1'
      if (!fileInfo||!operations) return;
      fileInfo.style.transform = `translateY(0)`;
      operations.style.visibility = "visible";
      clearTimeout(this.timer);
      this.timer = window.setTimeout(this.toMini, 3000);
    }

    timer: number = 0;

    showOperateMenu(e: MouseEvent) {
      dropDownMenu(e, [
        {label: "保存为播放列表", callback: this.saveToPlayList},
        {label: "清空正在播放", callback: this.clearPlayingList}
      ]);
    }

    saveToPlayList() {
      editPlayListModal({isRename: false}).then(name => {
        this.$store.dispatch("playList/createPlayList", {name, fileIds: this.playingList.map((o: File) => o.id)});
      });
    }

    clearPlayingList() {
      this.$store.dispatch('audio/abort')
      this.$store.commit('playList/setPlayingList',[])
    }

    showVolumeModal() {
      const that = this;
      // this.menuX = e.pageX - (<HTMLElement>this.$refs.PlayingToolBar).offsetLeft
      // this.menuY = e.pageY
      this.volumeModalShow = true;

      function hideVolumeModal(e: MouseEvent) {

        if (!isInSelf(<HTMLElement>e.target, <HTMLElement>that.$refs.volumeModalShow)) {
          e.stopPropagation();
          that.volumeModalShow = false;
          document.removeEventListener("click", hideVolumeModal, true);
        }
      }

      this.volumeModalShow = true;

      document.addEventListener("click", hideVolumeModal, true);
    }


    togglePlay() {
      this.$store.dispatch("audio/togglePlay");
    }

    formatProgress(val: number) {
      var sec = this.duration * (val / 100);
      return getTimeStr(sec);
    }


  }
</script>

<style scoped lang="scss">

  /*transition: transform .3s;*/
  .top {
    transform: translateY(0);
    /*animation: barToTop .5s forwards;*/
  }

  .moveTop {
    animation: barToTop .5s forwards;
  }

  .bottom {
    transform: translateY(calc(100vh - 100%))
    /*animation: barToBottom .6s;*/

  }

  .moveBottom {
    animation: barToBottom .6s;
  }

  .PlayingToolBar {

    .file-info {
      margin-top: 44px;
      display: flex;
      height: 120px;
      color: #fff;
      transition: transform .25s;
      .cover {
        width: 120px;
        height: 120px;
        img {
          width: 100%;
          height: 100%;
        }
      }
      .info {
        display: flex;
        flex-direction: column;
        justify-content: center;
        padding-right: 6px;
        text-align: left;
        margin-left: 10px;
        h5 {
          font-size: 30px;
        }
        p {
          margin: 0;
          font-size: 18px;
        }
      }
    }

    .progress {
      width: 100%;
      height: 50px;
      display: flex;
      align-items: center;
      margin-top: 20px;
      box-sizing: border-box;
      padding: 0 20px;
      .current, .total {
        font-size: 12px;
        color: #fff;
      }
      .current {
        margin-right: 10px;
      }
      .total {
        margin-left: 10px;
      }
      .slider {

        flex: 1;
      }
    }

    .buttons {
      display: flex;
      justify-content: space-between;
      height: 50px;

      .operate {

        color: #fff;
        /*        width: 35px;
                height:35px;*/
        display: inline-flex;
        justify-content: center;
        align-items: center;
        font-size: 30px;
        position: relative;
        border-radius: 50%;
        border: 1px solid rgba(255, 255, 255, 0);
        /*box-sizing: border-box;*/
        padding: 5px;

        margin: 0 10px 0 0;

        &.active {
          border: 1px solid rgba(255, 255, 255, .5);
        }

        &:hover {
          /*opacity: .4;*/
          color: rgba(255, 255, 255, .4);
        }

        &.loop {
          width: 24px;
          height: 24px;
          padding: 8px;
          box-sizing: content-box;

          &.active {
            border: 1px solid rgba(255, 255, 255, .5);
          }
          &:hover {
            opacity: .4;
          }
        }

        &.volume {
          padding: 0;
          width: 42px;
          height: 42px;
          position: relative;
          top: 4px;
          font-size: 38px;
        }
        &.play-list {
          font-size: 35px;
        }
        &.full-screen {
          font-size: 26px;
          position: relative;
          bottom: 3px;
        }
      }
    }

    .arrow {
      text-align: center;
      font-size: 28px;
      height: 30px;
      width: 100%;
      color: #fff;
      position: relative;
      bottom: 6px;
      & > i {
        transition: transform .3s;
      }
      & > img {
        transition: transform .3s;
        width: 16px;
        height: 16px;
      }
      &.up {
        & > i {
          transform: rotateZ(-180deg);
        }
        & > img {
          transform: rotateZ(-180deg);
        }
      }
    }
  }

  @keyframes barToBottom {
    0% {
      transform: translateY(0);
    }
    15% {
      transform: translateY(-10px);
    }
    100% {
      transform: translateY(calc(100vh - 100%))
    }
  }

  @keyframes barToTop {
    0% {
      transform: translateY(calc(100vh - 100%))
    }
    100% {
      transform: translateY(0);
    }
  }

  .volume-modal-enter, .volume-modal-leave-to {
    opacity: 0;
  }

  .volume-modal {
    transition: all .3s;
    width: 300px;
    height: 50px;
    background-color: #fff;
    display: flex;
    justify-content: center;
    align-items: center;
    position: absolute;
    top: -50px;

    .opts {
      width: 100%;
      height: 100%;
      box-sizing: border-box;
      display: flex;
      justify-content: center;
      align-items: center;
      padding: 0 40px 0 20px;

      .icon {
        font-size: 40px;
        width: 40px;
        margin-right: 10px;
        display: flex;
        justify-content: center;
        align-items: center;
        color: #333;
      }
      .slide {
        flex: 1;
        display: block;
      }
    }
  }


</style>


<style lang="scss">
  .loop > img {
    position: relative;
    bottom: 1px;
  }

</style>