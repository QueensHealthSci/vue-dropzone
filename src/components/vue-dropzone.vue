<template>
  <div :id="id" ref="dropzoneElement" :class="{ 'vue-dropzone dropzone': includeStyling }">
    <div class="dz-message">
      <i class="fa fa-cloud-upload"></i>
      <p class="ejs-m-b---d">Drop files here to upload</p>
      <EjsButton :small="true">Browse Files</EjsButton>
    </div>
  </div>
</template>

<script>
/* tslint:disable */
import Dropzone from "dropzone"; //eslint-disable-line
import awsEndpoint from "../services/urlsigner";
import { default as EjsButton } from "@/components/Button/Button.vue";

Dropzone.autoDiscover = false;

export default {
  components: {
    EjsButton
  },
  props: {
    id: {
      type: String,
      required: true,
      default: "dropzone"
    },
    options: {
      type: Object,
      required: true
    },
    includeStyling: {
      type: Boolean,
      default: true,
      required: false
    },
    awss3: {
      type: Object,
      required: false,
      default: null
    },
    destroyDropzone: {
      type: Boolean,
      default: true,
      required: false
    },
    duplicateCheck: {
      type: Boolean,
      default: false,
      required: false
    },
    useCustomSlot: {
      type: Boolean,
      default: false,
      required: false
    }
  },
  data() {
    return {
      isS3: false,
      isS3OverridesServerPropagation: false,
      wasQueueAutoProcess: true
    };
  },
  computed: {
    dropzoneSettings() {
      let defaultValues = {
        thumbnailWidth: 200,
        thumbnailHeight: 200
      };
      Object.keys(this.options).forEach(function(key) {
        defaultValues[key] = this.options[key];
      }, this);
      if (this.awss3 !== null) {
        defaultValues["autoProcessQueue"] = false;
        this.isS3 = true; //eslint-disable-line
        this.isS3OverridesServerPropagation =
          this.awss3.sendFileToServer === false; //eslint-disable-line
        if (this.options.autoProcessQueue !== undefined)
          this.wasQueueAutoProcess = this.options.autoProcessQueue; //eslint-disable-line

        if (this.isS3OverridesServerPropagation) {
          defaultValues["url"] = files => {
            return files[0].s3Url;
          };
        }
      }
      return defaultValues;
    }
  },
  mounted() {
    if (this.$isServer && this.hasBeenMounted) {
      return;
    }
    this.hasBeenMounted = true;

    this.dropzone = new Dropzone(
      this.$refs.dropzoneElement,
      this.dropzoneSettings
    );
    let vm = this;

    this.dropzone.on("thumbnail", function(file, dataUrl) {
      vm.$emit("vdropzone-thumbnail", file, dataUrl);
    });

    this.dropzone.on("addedfile", function(file) {
      var isDuplicate = false;
      if (vm.duplicateCheck) {
        if (this.files.length) {
          var _i, _len;
          for (
            _i = 0, _len = this.files.length;
            _i < _len - 1;
            _i++ // -1 to exclude current file
          ) {
            if (
              this.files[_i].name === file.name &&
              this.files[_i].size === file.size &&
              this.files[_i].lastModifiedDate.toString() ===
                file.lastModifiedDate.toString()
            ) {
              this.removeFile(file);
              isDuplicate = true;
              vm.$emit("vdropzone-duplicate-file", file);
            }
          }
        }
      }

      vm.$emit("vdropzone-file-added", file);
      if (vm.isS3 && vm.wasQueueAutoProcess && !file.manuallyAdded) {
        vm.getSignedAndUploadToS3(file);
      }
    });

    this.dropzone.on("addedfiles", function(files) {
      vm.$emit("vdropzone-files-added", files);
    });

    this.dropzone.on("removedfile", function(file) {
      vm.$emit("vdropzone-removed-file", file);
      if (file.manuallyAdded && vm.dropzone.options.maxFiles !== null)
        vm.dropzone.options.maxFiles++;
    });

    this.dropzone.on("success", function(file, response) {
      vm.$emit("vdropzone-success", file, response);
      if (vm.isS3) {
        if (vm.isS3OverridesServerPropagation) {
          var xmlResponse = new window.DOMParser().parseFromString(
            response,
            "text/xml"
          );
          var s3ObjectLocation = xmlResponse.firstChild.children[0].innerHTML;
          vm.$emit("vdropzone-s3-upload-success", s3ObjectLocation);
        }
        if (vm.wasQueueAutoProcess) vm.setOption("autoProcessQueue", false);
      }
    });

    this.dropzone.on("successmultiple", function(file, response) {
      vm.$emit("vdropzone-success-multiple", file, response);
    });

    this.dropzone.on("error", function(file, message, xhr) {
      vm.$emit("vdropzone-error", file, message, xhr);
      if (this.isS3) vm.$emit("vdropzone-s3-upload-error");
    });

    this.dropzone.on("errormultiple", function(files, message, xhr) {
      vm.$emit("vdropzone-error-multiple", files, message, xhr);
    });

    this.dropzone.on("sending", function(file, xhr, formData) {
      if (vm.isS3) {
        if (vm.isS3OverridesServerPropagation) {
          let signature = file.s3Signature;
          Object.keys(signature).forEach(function(key) {
            formData.append(key, signature[key]);
          });
        } else {
          formData.append("s3ObjectLocation", file.s3ObjectLocation);
        }
      }
      vm.$emit("vdropzone-sending", file, xhr, formData);
    });

    this.dropzone.on("sendingmultiple", function(file, xhr, formData) {
      vm.$emit("vdropzone-sending-multiple", file, xhr, formData);
    });

    this.dropzone.on("complete", function(file) {
      vm.$emit("vdropzone-complete", file);
    });

    this.dropzone.on("completemultiple", function(files) {
      vm.$emit("vdropzone-complete-multiple", files);
    });

    this.dropzone.on("canceled", function(file) {
      vm.$emit("vdropzone-canceled", file);
    });

    this.dropzone.on("canceledmultiple", function(files) {
      vm.$emit("vdropzone-canceled-multiple", files);
    });

    this.dropzone.on("maxfilesreached", function(files) {
      vm.$emit("vdropzone-max-files-reached", files);
    });

    this.dropzone.on("maxfilesexceeded", function(file) {
      vm.$emit("vdropzone-max-files-exceeded", file);
    });

    this.dropzone.on("processing", function(file) {
      vm.$emit("vdropzone-processing", file);
    });

    this.dropzone.on("processingmultiple", function(files) {
      vm.$emit("vdropzone-processing-multiple", files);
    });

    this.dropzone.on("uploadprogress", function(file, progress, bytesSent) {
      vm.$emit("vdropzone-upload-progress", file, progress, bytesSent);
    });

    this.dropzone.on("totaluploadprogress", function(
      totaluploadprogress,
      totalBytes,
      totalBytesSent
    ) {
      vm.$emit(
        "vdropzone-total-upload-progress",
        totaluploadprogress,
        totalBytes,
        totalBytesSent
      );
    });

    this.dropzone.on("reset", function() {
      vm.$emit("vdropzone-reset");
    });

    this.dropzone.on("queuecomplete", function() {
      vm.$emit("vdropzone-queue-complete");
    });

    this.dropzone.on("drop", function(event) {
      vm.$emit("vdropzone-drop", event);
    });

    this.dropzone.on("dragstart", function(event) {
      vm.$emit("vdropzone-drag-start", event);
    });

    this.dropzone.on("dragend", function(event) {
      vm.$emit("vdropzone-drag-end", event);
    });

    this.dropzone.on("dragenter", function(event) {
      vm.$emit("vdropzone-drag-enter", event);
    });

    this.dropzone.on("dragover", function(event) {
      vm.$emit("vdropzone-drag-over", event);
    });

    this.dropzone.on("dragleave", function(event) {
      vm.$emit("vdropzone-drag-leave", event);
    });

    vm.$emit("vdropzone-mounted");
  },
  beforeDestroy() {
    if (this.destroyDropzone) this.dropzone.destroy();
  },
  methods: {
    manuallyAddFile: function(file, fileUrl) {
      file.manuallyAdded = true;
      this.dropzone.emit("addedfile", file);
      let containsImageFileType = false;
      if (
        fileUrl.indexOf(".svg") > -1 ||
        fileUrl.indexOf(".png") > -1 ||
        fileUrl.indexOf(".jpg") > -1 ||
        fileUrl.indexOf(".jpeg") > -1 ||
        fileUrl.indexOf(".gif") > -1 ||
        fileUrl.indexOf(".webp") > -1
      )
        containsImageFileType = true;
      if (
        this.dropzone.options.createImageThumbnails &&
        containsImageFileType &&
        file.size <= this.dropzone.options.maxThumbnailFilesize * 1024 * 1024
      ) {
        fileUrl && this.dropzone.emit("thumbnail", file, fileUrl);

        var thumbnails = file.previewElement.querySelectorAll(
          "[data-dz-thumbnail]"
        );
        for (var i = 0; i < thumbnails.length; i++) {
          thumbnails[i].style.width =
            this.dropzoneSettings.thumbnailWidth + "px";
          thumbnails[i].style.height =
            this.dropzoneSettings.thumbnailHeight + "px";
          thumbnails[i].style["object-fit"] = "contain";
        }
      }
      this.dropzone.emit("complete", file);
      if (this.dropzone.options.maxFiles) this.dropzone.options.maxFiles--;
      this.dropzone.files.push(file);
      this.$emit("vdropzone-file-added-manually", file);
    },
    setOption: function(option, value) {
      this.dropzone.options[option] = value;
    },
    removeAllFiles: function(bool) {
      this.dropzone.removeAllFiles(bool);
    },
    processQueue: function() {
      let dropzoneEle = this.dropzone;
      if (this.isS3 && !this.wasQueueAutoProcess) {
        this.getQueuedFiles().forEach(file => {
          this.getSignedAndUploadToS3(file);
        });
      } else {
        this.dropzone.processQueue();
      }
      this.dropzone.on("success", function() {
        dropzoneEle.options.autoProcessQueue = true;
      });
      this.dropzone.on("queuecomplete", function() {
        dropzoneEle.options.autoProcessQueue = false;
      });
    },
    init: function() {
      return this.dropzone.init();
    },
    destroy: function() {
      return this.dropzone.destroy();
    },
    updateTotalUploadProgress: function() {
      return this.dropzone.updateTotalUploadProgress();
    },
    getFallbackForm: function() {
      return this.dropzone.getFallbackForm();
    },
    getExistingFallback: function() {
      return this.dropzone.getExistingFallback();
    },
    setupEventListeners: function() {
      return this.dropzone.setupEventListeners();
    },
    removeEventListeners: function() {
      return this.dropzone.removeEventListeners();
    },
    disable: function() {
      return this.dropzone.disable();
    },
    enable: function() {
      return this.dropzone.enable();
    },
    filesize: function(size) {
      return this.dropzone.filesize(size);
    },
    accept: function(file, done) {
      return this.dropzone.accept(file, done);
    },
    addFile: function(file) {
      return this.dropzone.addFile(file);
    },
    removeFile: function(file) {
      this.dropzone.removeFile(file);
    },
    getAcceptedFiles: function() {
      return this.dropzone.getAcceptedFiles();
    },
    getRejectedFiles: function() {
      return this.dropzone.getRejectedFiles();
    },
    getFilesWithStatus: function() {
      return this.dropzone.getFilesWithStatus();
    },
    getQueuedFiles: function() {
      return this.dropzone.getQueuedFiles();
    },
    getUploadingFiles: function() {
      return this.dropzone.getUploadingFiles();
    },
    getAddedFiles: function() {
      return this.dropzone.getAddedFiles();
    },
    getActiveFiles: function() {
      return this.dropzone.getActiveFiles();
    },
    getSignedAndUploadToS3(file) {
      var promise = awsEndpoint.sendFile(
        file,
        this.awss3,
        this.isS3OverridesServerPropagation
      );
      if (!this.isS3OverridesServerPropagation) {
        promise.then(response => {
          if (response.success) {
            file.s3ObjectLocation = response.message;
            setTimeout(() => this.dropzone.processFile(file));
            this.$emit("vdropzone-s3-upload-success", response.message);
          } else {
            if ("undefined" !== typeof response.message) {
              this.$emit("vdropzone-s3-upload-error", response.message);
            } else {
              this.$emit(
                "vdropzone-s3-upload-error",
                "Network Error : Could not send request to AWS. (Maybe CORS error)"
              );
            }
          }
        });
      } else {
        promise.then(() => {
          setTimeout(() => this.dropzone.processFile(file));
        });
      }
      promise.catch(error => {
        alert(error);
      });
    },
    setAWSSigningURL(location) {
      if (this.isS3) {
        this.awss3.signingURL = location;
      }
    }
  }
};
</script>

<style lang="scss">
@import "@/styles/_variables.scss";
@import "@/styles/_mixins.scss";

.vue-dropzone {
  align-items: center;
  background-color: ejsc($ejs-color-grey--light);
  border: 5px dashed ejsc($ejs-color-layout--border);
  border-radius: $ejs-border-radius--lg;
  display: flex;
  font-family: "Arial", sans-serif;
  justify-content: center;
  letter-spacing: 0.2px;
  transition: 0.2s linear;
  min-height: 276px;

  &:hover {
    background-color: ejsc($ejs-color-grey--medium);
  }

  .dz-message {
    display: flex;
    flex-direction: column;
    margin: 0;

    > i {
      color: ejsc($ejs-color-icon---d);
      font-size: $ejs-type-size--xl;
      margin-bottom: $ejs-spacing--xs;
    }

    > .button {
      align-self: center;
    }
  }

  > .dz-preview {
    margin: $ejs-spacing---d !important;

    &:hover {
      .dz-progress,
      .dz-success-mark,
      .dz-error-mark {
        opacity: 0;
      }
      
      .dz-remove {
        opacity: 1;
      }
    }

    .dz-image {
      border-radius: $ejs-border-radius--lg;
      // overflow: hidden;
      width: 100%;
      height: 100%;

      img:not([src]) {
        width: 200px;
        height: 200px;
      }

      &:hover img {
        transform: none;
        -webkit-filter: none;
      }
    }

    .dz-details {
      bottom: 0;
      top: 0;
      background-color: ejsc($ejs-color-icon---d);
      border-radius: $ejs-border-radius--lg;
      transition: opacity 0.2s linear;
      text-align: left;

      .dz-filename {
        overflow: hidden;

        span {
          background-color: transparent;
          color: ejsc($ejs-color--white);
          padding: 0.3em 0.4em;
        }

        &:not(:hover) span {
          border: none;
        }

        &:hover {
          overflow: visible;
          
          span {
            background-color: ejsc($ejs-color--black, 0.85);
            border: none;
          }
        }
      }

      .dz-size {
        span {
          background-color: transparent;
          color: ejsc($ejs-color--white);
        }

        strong {
          color: ejsc($ejs-color--white);
        }
      }
    }

    .dz-progress .dz-upload {
      background: #cccccc;
    }

    .dz-remove {
      position: absolute;
      z-index: 30;
      margin-left: 0;
      padding: 0;
      width: 42px;
      height: 42px;
      border-radius: 50%;
      display: flex;
      align-items: center;
      justify-content: center;
      top: 10px;
      bottom: inherit;
      right: 0;
      margin-right: 10px;
      text-decoration: none;
      text-transform: uppercase;
      font-size: 0.8rem;
      font-weight: 800;
      letter-spacing: 1.1px;
      opacity: 0;
      transition: 0.2s;

      > i {
        color: ejsc($ejs-color--white);
        cursor: pointer;
        font-size: 22px;

        &::before {
          content: "\f00d";
        }
      }

      &:hover {
        background-color: ejsc($ejs-color--black, 0.2);
        text-decoration: none;
      }
      
      &:focus {
        background-color: ejsc($ejs-color-state--focus);
        opacity: 1;
        text-decoration: none;
      }
    }

    .dz-success-mark,
    .dz-error-mark {
      margin-left: auto;
      margin-top: auto;
      width: 100%;
      top: 35%;
      left: 0;
      transition: 0.2s linear;

      svg {
        margin-left: auto;
        margin-right: auto;
      }
    }

    .dz-error-mark {
      margin-left: auto;
      margin-right: auto;
      left: 0;
      width: 100%;
      text-align: center;

      &::after {
        display: none;
      }
    }

    .dz-error-message {
      border-radius: $ejs-border-radius---d;
      bottom: 15px;
      top: initial;
      left: 15px;
      width: auto;
      background: ejsc($ejs-color--black, 0.2);

      span {
        color: ejsc($ejs-color--white);
      }
    }
  }
}
</style>
