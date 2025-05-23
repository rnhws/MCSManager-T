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
      plugin: () => [json(), ...(isJSON ? jsonLintExtensions : [])] as const
    },
    { 
      name: ["js", "jsx", "ts", "tsx", "mjs", "djs"], 
      plugin: () => [javascript({ jsx: true, typescript: ext === "ts" })] as const
    },
    { 
      name: ["xml"], 
      plugin: () => [xml()] as const
    },
    { 
      name: ["css", "less", "scss"], 
      plugin: () => [css()] as const
    },
    { 
      name: ["html", "vue"], 
      plugin: () => [html()] as const
    },
    { 
      name: ["yaml", "yml", "toml"], 
      plugin: () => [new LanguageSupport(StreamLanguage.define(yamlMode.yaml))] as const
    },
    { 
      name: ["properties", "ini"], 
      plugin: () => [new LanguageSupport(StreamLanguage.define(propertiesMode.properties))] as const
    },
    { 
      name: ["shell", "sh", "bat", "cmd"], 
      plugin: () => [new LanguageSupport(StreamLanguage.define(shellMode.shell))] as const
    },
    { 
      name: ["py", "pyi", "pyw"], 
      plugin: () => [python()] as const
    }
  ];

  const extensions = languagesMap.find(item => item.name.includes(ext))?.plugin() ?? [javascript()];
  return extensions.flat();
};

let editor: EditorView | null = null;

const createThemeExtension = () => EditorView.theme({
  "&": { 
    fontSize: `${currentFontSize.value}px`,
    lineHeight: `${currentLineHeight.value}px`,
    "--cm-lint-marker-error-color": "#dc3545"
  },
  ".cm-content": {
    fontSize: `${currentFontSize.value}px`,
    lineHeight: `${currentLineHeight.value}px`,
    "border-left": "1px solid #fbfbfb"
  },
  ".cm-gutters": {
    backgroundColor: "#1a1a1c",
    color: "#666672"
  }
});

const baseExtensions = [
  basicSetup,
  tokyoNight,
  EditorView.lineWrapping,
  EditorView.updateListener.of(update => {
    if (!update.changes.empty) {
      emit("update:text", update.state.doc.toString());
    }
  })
];

const updateEditor = () => {
  if (!editor) return;
  
  const fullExtensions = [
    ...baseExtensions,
    ...getLanguageExtension(),
    createThemeExtension()
  ];

  editor.dispatch({
    effects: StateEffect.reconfigure.of(fullExtensions)
  });
};

const initEditor = () => {
  const startState = EditorState.create({
    doc: props.text,
    extensions: [
      ...baseExtensions,
      ...getLanguageExtension(),
      createThemeExtension()
    ]
  });

  const parentElement = document.getElementById(DOM_ID);
  if (!parentElement) return;
  editor = new EditorView({ state: startState, parent: parentElement });
};

const handleTouchStart = (e: TouchEvent) => {
  if (e.touches.length === 2) {
    initialPinchDistance = Math.hypot(
      e.touches[1].clientX - e.touches[0].clientX,
      e.touches[1].clientY - e.touches[0].clientY
    );
  }
};

const handleTouchMove = debounce((e: TouchEvent) => {
  if (e.touches.length === 2 && editorContainer.value) {
    e.preventDefault();
    const currentDistance = Math.hypot(
      e.touches[1].clientX - e.touches[0].clientX,
      e.touches[1].clientY - e.touches[0].clientY
    );
    
    if (initialPinchDistance && currentDistance) {
      const scale = currentDistance / initialPinchDistance;
      const newScale = Math.min(Math.max(currentScale * scale, MIN_SCALE), MAX_SCALE);
      
      currentFontSize.value = Math.round(baseFontSize * newScale);
      currentLineHeight.value = Math.round(baseLineHeight * newScale);
      currentScale = newScale;
      
      updateEditor();
    }
    initialPinchDistance = currentDistance;
  }
}, 100);

const handleTouchEnd = () => {
  initialPinchDistance = null;
  baseFontSize = currentFontSize.value;
  baseLineHeight = currentLineHeight.value;
};

onMounted(initEditor);
onBeforeUnmount(() => editor?.destroy());
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
  touch-action: pan-y;

  @media (max-width: 768px) {
    height: 60vh;
    border-radius: 0;
  }
}

.editor-container {
  min-height: 100%;
  padding: 12px;
}

.file-editor {
  :deep(.cm-editor) {
    min-height: calc(v-bind('props.height') - 24px);
  }

  :deep(.cm-lintRange-error) {
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 6 3'%3E%3Cg fill='%23dc3545'%3E%3Cpolygon points='5.5,0 2.5,3 1.1,3 4.1,0'/%3E%3Cpolygon points='4,0 6,2 6,0.6 5.4,0'/%3E%3Cpolygon points='0,2 1,3 2.4,3 0,0.6'/%3E%3C/g%3E%3C/svg%3E");
    background-position: bottom left;
    background-repeat: repeat-x;
  }

  :deep(.cm-lint-marker-error) {
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='%23dc3545'%3E%3Cpath d='M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-2h2v2zm0-4h-2V7h2v6z'/%3E%3C/svg%3E");
    width: 18px;
    height: 18px;
  }
}
</style>