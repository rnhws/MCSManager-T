<script setup lang="ts">
import { ref, onBeforeUnmount, onMounted } from "vue";
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
import { EditorState, StateEffect } from "@codemirror/state";
import { tokyoNight } from "@uiw/codemirror-theme-tokyo-night";
import { getRandomId } from "@/tools/randId";
import * as yamlMode from "@codemirror/legacy-modes/mode/yaml";
import * as propertiesMode from "@codemirror/legacy-modes/mode/properties";
import * as shellMode from "@codemirror/legacy-modes/mode/shell";
import { useScreen } from "../hooks/useScreen";
import { useAppConfigStore } from "@/stores/useAppConfigStore";
import { linter } from "@codemirror/lint";
import { jsonParseLinter } from "@codemirror/lang-json";
import { lintGutter } from "@codemirror/lint";

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
let baseFontSize = isPhone.value ? 14 : 15;
let baseLineHeight = isPhone.value ? 22 : 24;
const currentFontSize = ref(baseFontSize);
const currentLineHeight = ref(baseLineHeight);

const jsonLintExtensions = [
  lintGutter(),
  linter(jsonParseLinter())
];

const getLanguageExtension = () => {
  const ext = getFileExtName(props.filename);
  const isJSON = ["json", "json5"].includes(ext);
  
  const languagesMap = [
    { 
      name: ["json", "json5"], 
      plugin: () => [json(), ...(isJSON ? jsonLintExtensions : [])] 
    },
    { name: ["js", "jsx", "ts", "tsx", "mjs", "djs"], plugin: () => [javascript({ jsx: true, typescript: ext === "ts" })] },
    { name: ["xml"], plugin: () => [xml()] },
    { name: ["css", "less", "scss"], plugin: () => [css()] },
    { name: ["html", "vue"], plugin: () => [html()] },
    { name: ["yaml", "yml", "toml"], plugin: () => [new LanguageSupport(StreamLanguage.define(yamlMode.yaml))] },
    { name: ["properties", "ini"], plugin: () => [new LanguageSupport(StreamLanguage.define(propertiesMode.properties))] },
    { name: ["shell", "sh", "bat", "cmd"], plugin: () => [new LanguageSupport(StreamLanguage.define(shellMode.shell))] },
    { name: ["py", "pyi", "pyw"], plugin: () => [python()] }
  ];

  return languagesMap.find(item => item.name.includes(ext))?.plugin() ?? [javascript()];
};

let editor: EditorView | null = null;

const baseExtensions = [
  basicSetup,
  tokyoNight,
  EditorView.lineWrapping,
  EditorView.updateListener.of(update => {
    if (!update.changes.empty) emit("update:text", update.state.doc.toString());
  })
];

const updateEditor = () => {
  if (!editor) return;
  editor.dispatch({
    effects: StateEffect.reconfigure.of([
      ...baseExtensions,
      ...getLanguageExtension(),
      EditorView.theme({
        "&": { 
          fontSize: `${Math.max(8, currentFontSize.value)}px`,
          lineHeight: `${currentLineHeight.value}px`,
          letterSpacing: `${Math.max(0, 0.12 * currentFontSize.value)}px`
        },
        ".cm-content": {
          fontSize: `${Math.max(8, currentFontSize.value)}px`,
          lineHeight: `${currentLineHeight.value}px`,
          minWidth: `${currentFontSize.value * 40}px`
        },
        ".cm-gutters": {
          fontSize: `${Math.max(8, currentFontSize.value)}px`,
          minWidth: `${currentFontSize.value * 4}px`
        }
      })
    ])
  });
  adjustContainerSize();
};

const initEditor = () => {
  const startState = EditorState.create({
    doc: props.text,
    extensions: [
      ...baseExtensions,
      ...getLanguageExtension(),
      EditorView.theme({
        "&": { 
          fontSize: `${currentFontSize.value}px`,
          lineHeight: `${currentLineHeight.value}px`
        }
      })
    ]
  });

  const parentElement = document.getElementById(DOM_ID);
  if (!parentElement) return;
  editor = new EditorView({ state: startState, parent: parentElement });
};

const handleTouchMove = (e: TouchEvent) => {
  if (e.touches.length === 2 && editorContainer.value) {
    e.preventDefault();
    const t1 = e.touches[0];
    const t2 = e.touches[1];
    const currentDistance = Math.hypot(t2.clientX - t1.clientX, t2.clientY - t1.clientY);
    
    if (initialPinchDistance && currentDistance) {
      const scale = currentDistance / initialPinchDistance;
      const newScale = Math.min(Math.max(currentScale * scale, MIN_SCALE), MAX_SCALE);
      
      if (Math.abs(newScale - currentScale) > 0.1) {
        currentFontSize.value = Math.max(8, Math.round(baseFontSize * newScale));
        currentLineHeight.value = Math.round(baseLineHeight * newScale);
        currentScale = newScale;
        requestAnimationFrame(updateEditor);
      }
    }
    initialPinchDistance = currentDistance;
  }
};

const handleTouchStart = (e: TouchEvent) => {
  if (e.touches.length === 2) {
    const t1 = e.touches[0];
    const t2 = e.touches[1];
    initialPinchDistance = Math.hypot(t2.clientX - t1.clientX, t2.clientY - t1.clientY);
    baseFontSize = currentFontSize.value;
    baseLineHeight = currentLineHeight.value;
  }
};

const handleTouchEnd = () => {
  initialPinchDistance = null;
  baseFontSize = currentFontSize.value;
  baseLineHeight = currentLineHeight.value;
};

const adjustContainerSize = () => {
  const container = editorContainer.value;
  if (container) {
    container.style.minWidth = `${currentFontSize.value * 60}px`;
    container.style.minHeight = `${currentFontSize.value * 30}px`;
  }
};

onMounted(() => {
  initEditor();
  const container = editorContainer.value;
  if (container) {
    container.addEventListener('touchstart', handleTouchStart);
    container.addEventListener('touchmove', handleTouchMove, { passive: false });
    container.addEventListener('touchend', handleTouchEnd);
    container.addEventListener('touchcancel', handleTouchEnd);
  }
});

onBeforeUnmount(() => {
  const container = editorContainer.value;
  if (container) {
    container.removeEventListener('touchstart', handleTouchStart);
    container.removeEventListener('touchmove', handleTouchMove);
    container.removeEventListener('touchend', handleTouchEnd);
    container.removeEventListener('touchcancel', handleTouchEnd);
  }
  editor?.destroy();
});
</script>

<template>
  <div class="editor-wrapper">
    <div class="editor-container" ref="editorContainer">
      <div :id="DOM_ID" class="file-editor"></div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.editor-wrapper {
  height: v-bind('props.height');
  overflow: auto;
  background: #1e1e1e;
  border-radius: 6px;
  touch-action: none;
  overscroll-behavior: contain;

  @media (max-width: 768px) {
    height: 60vh;
    border-radius: 0;
  }
}

.editor-container {
  min-height: 100%;
  padding: 12px;
  transform-origin: 0 0;
  transition: min-width 0.2s ease, min-height 0.2s ease;
  min-width: v-bind('currentFontSize * 60 + "px"');
}

.file-editor {
  :deep(.cm-editor) {
    min-height: calc(v-bind('props.height') - 24px);
    transition: font-size 0.15s cubic-bezier(0.4, 0, 0.2, 1), 
                line-height 0.15s cubic-bezier(0.4, 0, 0.2, 1);
  }

  :deep(.cm-content) {
    word-spacing: 0.05em;
    white-space: pre-wrap !important;
    overflow-wrap: anywhere;
    padding-right: 1.2em;
  }

  :deep(.cm-line) {
    min-height: calc(v-bind('currentLineHeight') * 1.2px);
  }

  :deep(.cm-gutters) {
    min-width: v-bind('currentFontSize * 4 + "px"') !important;
  }

  :deep(.cm-lint-marker-error) {
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='%23dc3545'%3E%3Cpath d='M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-2h2v2zm0-4h-2V7h2v6z'/%3E%3C/svg%3E");
    width: calc(1em * 1.3);
    height: calc(1em * 1.3);
    background-size: contain;
    margin-left: 0.3em;
  }

  :deep(.cm-lintRange-error) {
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 6 3'%3E%3Cg fill='%23dc3545'%3E%3Cpolygon points='5.5,0 2.5,3 1.1,3 4.1,0'/%3E%3Cpolygon points='4,0 6,2 6,0.6 5.4,0'/%3E%3Cpolygon points='0,2 1,3 2.4,3 0,0.6'/%3E%3C/g%3E%3C/svg%3E");
    background-position: bottom left;
    background-repeat: repeat-x;
    background-size: auto calc(0.18em + 1px);
  }
}
</style>