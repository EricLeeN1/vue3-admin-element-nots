<template>
  <div
    :class="{ fullscreen: fullscreen }"
    class="tinymce-container"
    :style="{ width: containerWidth }"
  >
    <textarea :id="tinymceId" class="tinymce-textarea" />
    <div class="editor-custom-btn-container">
      <editorImage color="#1890ff" class="editor-upload-btn" @successCBK="imageSuccessCBK" />
    </div>
  </div>
</template>

<script>
/**
 * docs:
 * https://panjiachen.github.io/vue-element-admin-site/feature/component/rich-editor.html#tinymce
 */
import {
  reactive,
  toRefs,
  computed,
  watch,
  nextTick,
  getCurrentInstance,
  onDeactivated,
  onActivated,
  onMounted,
  onUnmounted
} from 'vue'
import { ElMessage } from 'element-plus'
import editorImage from './components/EditorImage.vue'
import plugins from './plugins'
import toolbar from './toolbar'
import load from './dynamicLoadScript'

// TODO Add富文本编辑项目
// why use this cdn, detail see https://github.com/PanJiaChen/tinymce-all-in-one
const tinymceCDN = 'https://cdn.jsdelivr.net/npm/tinymce-all-in-one@4.9.3/tinymce.min.js'
export default {
  name: 'Tinymce',
  components: { editorImage },
  props: {
    id: {
      type: String,
      default() {
        return `vue-tinymce-${+new Date()}${(Math.random() * 1000).toFixed(0)}`
      }
    },
    modelValue: {
      type: String,
      default: ''
    },
    toolbar: {
      type: Array,
      required: false,
      default() {
        return []
      }
    },
    menubar: {
      type: String,
      default: 'file edit insert view format table'
    },
    height: {
      type: [Number, String],
      required: false,
      default: 360
    },
    width: {
      type: [Number, String],
      required: false,
      default: 'auto'
    }
  },
  emits: ['update:modelValue'],
  setup(props, { emit }) {
    const state = reactive({
      hasChange: false,
      hasInit: false,
      tinymceId: props.id,
      fullscreen: false,
      languageTypeList: {
        en: 'en',
        zh: 'zh_CN',
        es: 'es_MX',
        ja: 'ja'
      }
    })
    const { ctx } = getCurrentInstance()
    const containerWidth = computed(() => {
      const { width } = props
      if (/^[\d]+(\.[\d]+)?$/.test(width)) {
        // matches `100`, `'100'`
        return `${width}px`
      }
      return width
    })
    watch(
      () => props.modelValue,
      (val) => {
        if (!state.hasChange && state.hasInit) {
          nextTick(() => {
            window.tinymce.get(state.tinymceId).setContent(val || '')
          })
        }
      }
    )

    const initTinymce = () => {
      // eslint-disable-next-line no-underscore-dangle
      const _this = ctx
      window.tinymce.init({
        selector: `#${state.tinymceId}`,
        language: state.languageTypeList.en,
        height: props.height,
        body_class: 'panel-body ',
        object_resizing: false,
        toolbar: props.toolbar.length > 0 ? props.toolbar : toolbar,
        menubar: props.menubar,
        plugins,
        end_container_on_empty_block: true,
        powerpaste_word_import: 'clean',
        code_dialog_height: 450,
        code_dialog_width: 1000,
        advlist_bullet_styles: 'square',
        advlist_number_styles: 'default',
        imagetools_cors_hosts: ['www.tinymce.com', 'codepen.io'],
        default_link_target: '_blank',
        link_title: false,
        nonbreaking_force_tab: true, // inserting nonbreaking space &nbsp; need Nonbreaking Space Plugin
        init_instance_callback: (editor) => {
          if (_this.modelValue) {
            editor.setContent(_this.modelValue)
          }
          _this.hasInit = true
          editor.on('NodeChange Change KeyUp SetContent', () => {
            state.hasChange = true
            emit('update:modelValue', editor.getContent())
          })
        },
        setup(editor) {
          editor.on('FullscreenStateChanged', (e) => {
            _this.fullscreen = e.state
          })
        },
        // it will try to keep these URLs intact
        // https://www.tiny.cloud/docs-3x/reference/configuration/Configuration3x@convert_urls/
        // https://stackoverflow.com/questions/5196205/disable-tinymce-absolute-to-relative-url-conversions
        convert_urls: false
        // 整合七牛上传
        // images_dataimg_filter(img) {
        //   setTimeout(() => {
        //     const $image = $(img);
        //     $image.removeAttr('width');
        //     $image.removeAttr('height');
        //     if ($image[0].height && $image[0].width) {
        //       $image.attr('data-wscntype', 'image');
        //       $image.attr('data-wscnh', $image[0].height);
        //       $image.attr('data-wscnw', $image[0].width);
        //       $image.addClass('wscnph');
        //     }
        //   }, 0);
        //   return img
        // },
        // images_upload_handler(blobInfo, success, failure, progress) {
        //   progress(0);
        //   const token = _this.$store.getters.token;
        //   getToken(token).then(response => {
        //     const url = response.data.qiniu_url;
        //     const formData = new FormData();
        //     formData.append('token', response.data.qiniu_token);
        //     formData.append('key', response.data.qiniu_key);
        //     formData.append('file', blobInfo.blob(), url);
        //     upload(formData).then(() => {
        //       success(url);
        //       progress(100);
        //     })
        //   }).catch(err => {
        //     failure('出现未知问题，刷新页面，或者联系程序员')
        //     console.log(err);
        //   });
        // },
      })
    }
    const init = () => {
      // dynamic load tinymce from cdn
      load(tinymceCDN, (err) => {
        if (err) {
          ElMessage.error(err.message)
          return
        }
        initTinymce()
      })
    }

    const destroyTinymce = () => {
      const tinymce = window.tinymce.get(state.tinymceId)
      if (state.fullscreen) {
        tinymce.execCommand('mceFullScreen')
      }

      if (tinymce) {
        tinymce.destroy()
      }
    }
    const setContent = (value) => {
      window.tinymce.get(state.tinymceId).setContent(value)
    }
    const getContent = () => {
      window.tinymce.get(state.tinymceId).getContent()
    }
    onMounted(() => {
      init()
    })
    onUnmounted(() => {
      destroyTinymce()
    })
    onActivated(() => {
      if (window.tinymce) {
        initTinymce()
      }
    })
    onDeactivated(() => {
      destroyTinymce()
    })
    const imageSuccessCBK = (arr) => {
      arr.forEach((v) =>
        window.tinymce.get(state.tinymceId).insertContent(`<img class="wscnph" src="${v.url}" >`)
      )
    }
    return {
      ...toRefs(state),
      containerWidth,
      init,
      initTinymce,
      destroyTinymce,
      setContent,
      getContent,
      imageSuccessCBK
    }
  }
}
</script>

<style lang="scss" scoped>
.tinymce-container {
  position: relative;
  line-height: normal;
}

.tinymce-container {
  ::v-deep {
    .mce-fullscreen {
      z-index: 10000;
    }
  }
}

.tinymce-textarea {
  visibility: hidden;
  z-index: -1;
}

.editor-custom-btn-container {
  position: absolute;
  right: 4px;
  top: 4px;
  /*z-index: 2005;*/
}

.fullscreen .editor-custom-btn-container {
  z-index: 10000;
  position: fixed;
}

.editor-upload-btn {
  display: inline-block;
}
</style>
