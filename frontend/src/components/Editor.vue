<script setup lang="ts">
import { ref, onBeforeUnmount, onMounted } from "vue";
import { t } from "@/lang/i18n";
import { EditorView } from "@codemirror/view";
import { basicSetup } from "codemirror";
import { StreamLanguage, LanguageSupport } from "@codemirror/language";
import { javascript } from "@codemirror/lang-javascript";
import { python } from "@codemirror/lang-python";
import { json } from "@codemirror/lang-json";
import { css } from "@codemirror/lang-css";
import { html } from "@codemirror/lang-html";
import { xml } from "@codemirror/lang-xml";
import { getFileExtName } from "@/tools/fileManager";
import { EditorState, Compartment } from "@codemirror/state";
import { tokyoNight } from '@uiw/codemirror-theme-tokyo-night';
import { getRandomId } from "@/tools/randId";
import * as yamlMode from "@codemirror/legacy-modes/mode/yaml";
import * as propertiesMode from "@codemirror/legacy-modes/mode/properties";
import * as shellMode from "@codemirror/legacy-modes/mode/shell";
import { useScreen } from "../hooks/useScreen";
import { useAppConfigStore } from "@/stores/useAppConfigStore";
import { linter } from "@codemirror/lint";
import { jsonParseLinter } from "@codemirror/lang-json";
import { lintGutter } from "@codemirror/lint";
import { debounce } from "lodash-es";

const { isDarkTheme } = useAppConfigStore();
const emit = defineEmits(["update:text"]);
const uuid = getRandomId();
const DOM_ID = `file-editor-${uuid}`;
const { isPhone } = useScreen();

const props = defineProps<{
  text: string;
  height: string;
  filename: string;
}>();

const editorContainer = ref<HTMLElement>();
let initialPinchDistance: number | null = null;
let currentScale = 1;
const MAX_SCALE = 3;
const MIN_SCALE = 0.5;

const baseFontSize = isPhone.value ? "14px" : "15px";
const lineHeight = isPhone.value ? "22px" : "24px";

const getLanguageExtension = () => {
  const ext = getFileExtName(props.filename);
  const languagesMap = [
    {
      name: ["js", "jsx", "ts", "tsx", "mjs", "djs"],
      plugin: () => javascript({ jsx: true, typescript: ext === "ts" })
    },
    {
      name: ["json", "json5"],
      plugin: () => [
        json(),
        lintGutter(),
        linter(jsonParseLinter())
      ]
    },
    {
      name: ["xml"],
      plugin: () => xml()
    },
    {
      name: ["css", "less", "scss"],
      plugin: () => css()
    },
    {
      name: ["html", "vue"],
      plugin: () => html()
    },
    {
      name: ["yaml", "yml", "toml"],
      plugin: () => new LanguageSupport(StreamLanguage.define(yamlMode.yaml))
    },
    {
      name: ["properties", "ini"],
      plugin: () => new LanguageSupport(StreamLanguage.define(propertiesMode.properties))
    },
    {
      name: ["shell", "sh", "bat", "cmd"],
      plugin: () => new LanguageSupport(StreamLanguage.define(shellMode.shell))
    },
    {
      name: ["py", "pyi", "pyw"],
      plugin: () => python()
    }
  ];

  return languagesMap.find(item => item.name.includes(ext))?.plugin() ?? javascript();
};

let editor: EditorView | null = null;

const initEditor = () => {
  const startState = EditorState.create({
    doc: props.text,
    extensions: [
      basicSetup,
      tokyoNight,
      getLanguageExtension(),
      EditorView.lineWrapping,
      EditorView.theme({
        "&": {
          fontSize: baseFontSize,
          lineHeight: lineHeight
        },
        ".cm-content": {
          fontSize: baseFontSize,
          lineHeight: lineHeight,
          "border-left": "1px solid #fbfbfb"
        },
        ".cm-gutters": {
          backgroundColor: "#1a1a1c",
          color: "#666672"
        },
        ".cm-gutterElement": {
          justifyContent: "flex-end"
        },
        ".cm-activeLineGutter": {
          backgroundColor: "#2a2a2e",
          color: "#fff"
        },
        ".cm-lineNumbers .cm-gutterElement": {
          fontFamily: "Menlo, Monaco, Consolas, 'Courier New', monospace"
        },
        ".cm-lintRange-error": {
          backgroundImage: "url(\"data:image/svg+xml,%3Csvg%20xmlns%3D'http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg'%20viewBox%3D'0%200%206%203'%20enable-background%3D'new%200%200%206%203'%20height%3D'3'%20width%3D'6'%3E%3Cg%20fill%3D'%23dc3545'%3E%3Cpolygon%20points%3D'5.5%2C0%202.5%2C3%201.1%2C3%204.1%2C0'%2F%3E%3Cpolygon%20points%3D'4%2C0%206%2C2%206%2C0.6%205.4%2C0'%2F%3E%3Cpolygon%20points%3D'0%2C2%201%2C3%202.4%2C3%200%2C0.6'%2F%3E%3C%2Fg%3E%3C%2Fsvg%3E\")",
          backgroundRepeat: "repeat-x",
          backgroundPosition: "bottom left"
        }
      }),
      EditorView.updateListener.of((update) => {
        if (!update.changes.empty) {
          emit("update:text", update.state.doc.toString());
        }
      })
    ]
  });

  const parentElement = document.getElementById(DOM_ID);
  if (!parentElement) {
    console.error(`Editor container #${DOM_ID} not found.`);
    return;
  }
  editor = new EditorView({
    state: startState,
    parent: parentElement,
  });
};

const handleTouchStart = (e: TouchEvent) => {
  if (e.touches.length === 2) {
    initialPinchDistance = getDistance(e.touches[0], e.touches[1]);
  }
};

const handleTouchMove = debounce((e: TouchEvent) => {
  if (e.touches.length === 2 && editorContainer.value) {
    e.preventDefault();
    const currentDistance = getDistance(e.touches[0], e.touches[1]);
    
    if (initialPinchDistance && currentDistance) {
      const scale = currentDistance / initialPinchDistance;
      const newScale = Math.min(Math.max(currentScale * scale, MIN_SCALE), MAX_SCALE);
      
      if (newScale !== currentScale) {
        currentScale = newScale;
        applyZoom(newScale);
      }
    }
    initialPinchDistance = currentDistance;
  }
}, 10);

const handleTouchEnd = () => {
  initialPinchDistance = null;
};

const getDistance = (t1: Touch, t2: Touch) => {
  return Math.hypot(
    t2.clientX - t1.clientX,
    t2.clientY - t1.clientY
  );
};

const applyZoom = (scale: number) => {
  if (!editorContainer.value) return;
  
  editorContainer.value.style.transform = `scale(${scale})`;
  editorContainer.value.style.transformOrigin = '0 0';
  editorContainer.value.style.width = `${100 / scale}%`;
  editorContainer.value.style.height = `${100 / scale}%`;
};

onMounted(() => {
  initEditor();
  const container = editorContainer.value;
  if (container) {
    container.addEventListener('touchstart', handleTouchStart);
    container.addEventListener('touchmove', handleTouchMove, { passive: false });
    container.addEventListener('touchend', handleTouchEnd);
  }
});

onBeforeUnmount(() => {
  const container = editorContainer.value;
  if (container) {
    container.removeEventListener('touchstart', handleTouchStart);
    container.removeEventListener('touchmove', handleTouchMove);
    container.removeEventListener('touchend', handleTouchEnd);
  }
  editor?.destroy();
});
</script>

<template>
  <div class="editor-container" ref="editorContainer">
    <div :id="DOM_ID" class="file-editor"></div>
  </div>
</template>

<style lang="scss" scoped>
.editor-container {
  position: relative;
  height: v-bind('props.height');
  display: flex;
  flex-direction: column;
  background: #1e1e1e;
  border-radius: 6px;
  overflow: hidden;
  touch-action: none;
  transition: transform 0.1s ease-out;
  
  @media (max-width: 768px) {
    height: 60vh;
  }
}

.file-editor {
  flex: 1;
  overflow: hidden;
  -webkit-overflow-scrolling: touch;
  transform: translateZ(0);
  
  @media (max-width: 768px) {
    .cm-content {
      min-height: calc(100% + 100px);
      padding: 0 8px 0 8px!important;
    }
    .cm-gutters {
      font-size: 13px!important;
    }
  }
}

@media (max-width: 768px) {
  .editor-container {
    border-radius: 0;
  }
  
  .file-editor {
    font-size: 14px !important;
    line-height: 22px !important;
    
    .cm-gutters {
      font-size: 13px !important;
    }
  }
}
</style>