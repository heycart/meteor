<template>
  <!-- eslint-disable-next-line vuejs-accessibility/mouse-events-have-key-events -->
  <button type="button">
    <mt-icon
      v-tooltip="{
        message: tooltipText,
        width: 220,
        position: 'top',
        showDelay: 300,
        hideDelay: 0,
      }"
      class="mt-field-copyable"
      name="regular-copy"
      size="18"
      @click="copyToClipboard"
      @mouseleave="wasCopied = false"
    />
  </button>
</template>

<script lang="ts">
import { computed, defineComponent, ref } from "vue";
import MtIcon from "../../../icons-media/mt-icon/mt-icon.vue";
import MtTooltipDirective from "../../../../directives/tooltip.directive";
import MtNotificationMixin from "../../../../mixins/notification.mixin";
import { copyToClipboard as copyToClipboardUtil } from "../../../../utils/dom";
import { useI18n } from "vue-i18n";

export default defineComponent({
  name: "MtFieldCopyable",

  directives: {
    tooltip: MtTooltipDirective,
  },

  components: {
    "mt-icon": MtIcon,
  },

  mixins: [MtNotificationMixin],

  props: {
    copyableText: {
      type: String,
      required: false,
      default: null,
    },

    tooltip: {
      type: Boolean,
      required: false,
      default: false,
    },
  },

  setup(props) {
    const wasCopied = ref(false);

    const { t } = useI18n({
      messages: {
        en: {
          tooltip: {
            wasCopied: "Copied to clipboard.",
            canCopy: "Copy to clipboard.",
            notificationCopySuccessMessage: "Text has been copied to clipboard.",
            notificationCopyFailureMessage: "Text could not be copied to clipboard.",
            errorTitle: "Error copying to clipboard",
          },
        },
        zh: {
          tooltip: {
            wasCopied: "已复制到剪贴板。",
            canCopy: "复制到剪贴板。",
            notificationCopySuccessMessage: "文本已成功复制到剪贴板。",
            notificationCopyFailureMessage: "无法将文本复制到剪贴板。",
            errorTitle: "复制到剪贴板时出错",
          },
        },
      },
    });

    const tooltipText = computed(() =>
      wasCopied.value ? t("tooltip.wasCopied") : t("tooltip.canCopy"),
    );

    function copyToClipboard() {
      if (!props.copyableText) return;
      copyToClipboardUtil(props.copyableText);

      if (props.tooltip) {
        wasCopied.value = true;
      }
    }

    return {
      copyToClipboard,
      tooltipText,
      wasCopied,
    };
  },
});
</script>

<style scoped>
.mt-field-copyable {
  cursor: pointer;
}
</style>
