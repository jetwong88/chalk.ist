<script setup lang="ts">
import { useElementSize, useEventListener } from "@vueuse/core";
import {
  transformerCompactLineOptions,
  transformerNotationDiff,
  transformerNotationFocus,
} from "@shikijs/transformers";
import { bundledLanguages, type ShikiTransformer } from "shiki";
import { computed, h, ref, watch } from "vue";
import { store } from "~/lib/store";
import { BlockType } from "~/lib/enums";
import { createTheme } from "~/lib/create-theme";
import { useShiki } from "~/lib/shiki";
import type { CodeBlock } from "~/types";
import Annotation from "~/components/ui/editor/Annotation.vue";
import { state } from "~/lib/state";

const props = defineProps<{
  block: CodeBlock;
}>();

const { shiki, loadLanguage, isLanguageLoaded, isLanguageLoading, isReady } =
  useShiki();
const editor = ref<HTMLTextAreaElement>();
const formatted = ref<HTMLDivElement>();
const hasLanguage = ref(false);

function transformerLineNumbers(): ShikiTransformer {
  return {
    name: "line-number",
    line(line, index) {
      line.properties = {
        ...line.properties,
        class: ["line"],
        "data-line": `${index}`,
      };
      line.children.unshift({
        type: "element",
        tagName: "span",
        properties: {
          class: ["line-number"],
        },
        children: [
          {
            type: "text",
            value: `${index}`,
          },
        ],
      });
      return line;
    },
  };
}

function transformerAnnotations(
  transformations: { type: string; line: number; character?: number }[],
): ShikiTransformer {
  return {
    name: "annotations",
    line(line, index) {
      const lineTransformations = transformations.filter(
        (item) => item.line === index && item.type === "annotate",
      );
      if (lineTransformations.length === 0) return line;

      const annotations = lineTransformations.map(
        (item) =>
          ({
            type: "element",
            tagName: "Annotation",
            properties: {
              start: item.character,
              end: item.character,
              onRemove: (() => {
                const transformationIndex =
                  props.block.transformations.findIndex(
                    (item) =>
                      item.type === "annotate" &&
                      item.line === index &&
                      item.character === item.character,
                  );
                if (transformationIndex !== -1) {
                  props.block.transformations.splice(transformationIndex, 1);
                }
              }) as any,
              class: ["annotation", item.type],
            },
            children: [],
          }) satisfies typeof line,
      );

      const annotationContainer = {
        type: "element",
        tagName: "div",
        properties: {
          class: ["annotations"],
        },
        children: annotations,
      } satisfies typeof line;

      return {
        type: "element",
        tagName: "div",
        properties: {
          class: ["contents"],
        },
        children: [line, annotationContainer],
      };
    },
  };
}

watch(
  [() => props.block.language, shiki, isReady],
  async () => {
    if (!shiki.value || !isReady.value) return;
    hasLanguage.value = false;

    try {
      if (!isLanguageLoaded(props.block.language)) {
        await loadLanguage(props.block.language);
      }
      hasLanguage.value = true;
    } catch (err) {
      console.error("Failed to load language:", err);
      hasLanguage.value = false;
    }
  },
  { immediate: true },
);

const isLoading = computed(() => {
  return !isReady.value || isLanguageLoading(props.block.language);
});

const shikiContent = computed(() => {
  if (
    !shiki.value ||
    !props.block ||
    props.block.type !== BlockType.Code ||
    !hasLanguage.value ||
    isLoading.value
  )
    return "";
  const classNames = ["shiki"];
  if (props.block.transformations.some((item) => item.type === "focus")) {
    classNames.push("has-focus");
  }
  if (store.value.showLineNumbers) {
    classNames.push("show-line-numbers");
  }
  const lineOptions = props.block.transformations.reduce(
    (mergedOptions, item) => {
      const existingOption = mergedOptions.find(
        (option) => option.line === item.line,
      );
      if (existingOption) {
        existingOption.classes.push(item.type);
      } else {
        mergedOptions.push({
          line: item.line,
          classes: [item.type],
        });
      }
      return mergedOptions;
    },
    [] as { line: number; classes: string[] }[],
  );
  const hast = shiki.value?.codeToHast(props.block.content, {
    lang: props.block.language,
    theme: store.value.useCustomTheme
      ? createTheme("Custom", store.value.customTheme)
      : store.value.colorTheme,
    transformers: [
      transformerNotationDiff(),
      transformerNotationFocus(),
      transformerLineNumbers(),
      transformerAnnotations(props.block.transformations),
      transformerCompactLineOptions(lineOptions),
    ],
    meta: {
      class: classNames.join(" "),
      tabindex: "-1",
    },
  });

  return shikiHastToVueH(hast);
});

const components: { [key: string]: any } = {
  Annotation,
};

function shikiHastToVueH(node: any) {
  if (node.type === "root") {
    return node.children.map(shikiHastToVueH);
  }
  if (node.type === "text") {
    return node.value;
  }
  if (node.type === "element") {
    const children = node.children.map(shikiHastToVueH);
    if (node.tagName in components) {
      return h(components[node.tagName], node.properties, () => children);
    }
    return h(node.tagName, node.properties, children);
  }
  return "";
}

useEventListener(editor, "keydown", async (e) => {
  if (!editor.value) return;
  const key = e.key;
  const metaKey = e.metaKey;
  const shiftKey = e.shiftKey;
  const textarea = editor.value;
  const tabSize = 2; // Change this according to your preference
  const val = textarea.value;
  const start = textarea.selectionStart;
  const end = textarea.selectionEnd;

  const startOfLine = val.lastIndexOf("\n", start - 1) + 1;
  let endOfLine = val.indexOf("\n", end);
  if (endOfLine === -1) endOfLine = val.length;

  if (key === "Tab") {
    e.preventDefault();

    if (shiftKey) {
      // Handle shift + tab to de-indent
      if (val.substring(start - tabSize, start) === " ".repeat(tabSize)) {
        textarea.value =
          val.substring(0, start - tabSize) + val.substring(start);
        textarea.selectionStart = textarea.selectionEnd = start - tabSize;
      }
    } else {
      // Handle tab to indent
      textarea.value =
        val.substring(0, start) + " ".repeat(tabSize) + val.substring(end);
      textarea.selectionStart = textarea.selectionEnd = start + tabSize;
    }
    textarea.dispatchEvent(new Event("input"));
  } else if (metaKey && (key === "]" || key === "[")) {
    e.preventDefault();
    if (key === "]") {
      // Handle CMD + ] to indent
      textarea.value =
        val.substring(0, startOfLine) +
        " ".repeat(tabSize) +
        val.substring(startOfLine);
      textarea.selectionStart = textarea.selectionEnd = start + tabSize; // Shifting cursor to the right
    } else if (key === "[") {
      // Handle CMD + [ to de-indent
      if (
        val.substring(startOfLine, startOfLine + tabSize) ===
        " ".repeat(tabSize)
      ) {
        textarea.value =
          val.substring(0, startOfLine) + val.substring(startOfLine + tabSize);
        textarea.selectionStart = textarea.selectionEnd = start - tabSize; // Shifting cursor to the left
      }
    }
    textarea.dispatchEvent(new Event("input"));
  }
});

const gutter = computed(() => {
  const base = store.value.innerPaddingX;
  const len = props.block.content.split("\n").length;
  if (!store.value.showLineNumbers) return `${base}px`;
  return len >= 100
    ? `calc(4ch + 4px + ${base}px)`
    : len >= 10
      ? `calc(3ch + 4px + ${base}px)`
      : `calc(2ch + 4px + ${base}px)`;
});

const fontFeatureSettings = computed(() => {
  if (!store.value.fontLigatures) return '"liga" 0, "calt" 0';
  return '"liga", "calt"';
});

const fontFamily = computed(() => {
  return `"${store.value.fontFamily}", Menlo, Monaco, "Courier New", monospace`;
});

const characterWidth = computed(() => {
  const el = document.createElement("div");
  el.style.fontFamily = fontFamily.value;
  el.style.fontSize = "13px";
  el.style.position = "absolute";
  el.style.opacity = "0";
  el.id = Math.random().toString(16).slice(2);
  el.innerText = "a";
  document.body.appendChild(el);
  const width = el.getBoundingClientRect().width;
  document.body.removeChild(el);
  return width;
});

useEventListener(formatted, "click", (event) => {
  const el = (event.target as HTMLElement).closest(".line") as HTMLSpanElement;
  if (state.editMode === "code") return;
  if (!el?.classList?.contains("line")) {
    return;
  }
  const line = parseInt(el.dataset.line!);
  const index = props.block.transformations.findIndex(
    (item) => item.type === state.editMode && item.line === line,
  );
  if (index === -1) {
    if (state.editMode === "add") {
      const index = props.block.transformations.findIndex(
        (item) => item.type === "remove" && item.line === line,
      );
      if (index !== -1) {
        props.block.transformations.splice(index, 1);
      }
    } else if (state.editMode === "remove") {
      const index = props.block.transformations.findIndex(
        (item) => item.type === "add" && item.line === line,
      );
      if (index !== -1) {
        props.block.transformations.splice(index, 1);
      }
    } else if (state.editMode === "highlight") {
      // remove "add" and "remove" transformations
      const index = props.block.transformations.findIndex(
        (item) => ["add", "remove"].includes(item.type) && item.line === line,
      );
      if (index !== -1) {
        props.block.transformations.splice(index, 1);
      }
    }
    props.block.transformations.push({
      type: state.editMode,
      line,
    });
  } else {
    props.block.transformations.splice(index, 1);
  }
});

function addCharacterTransformer(line: number, character: number) {
  const index = props.block.transformations.findIndex(
    (item) =>
      item.type === "annotate" &&
      item.line === line &&
      item.character === character,
  );
  if (index === -1) {
    props.block.transformations.push({
      type: "annotate",
      line,
      character,
    });
  } else {
    props.block.transformations.splice(index, 1);
  }
}

const { width: editorWidth } = useElementSize(editor);
const charactersPerLine = computed(() =>
  Math.floor(editorWidth.value / characterWidth.value),
);

const innerPaddingX = computed(() => `${store.value.innerPaddingX}px`);
</script>

<template>
  <div class="grid px-px [grid-template:1fr/1fr]">
    <div
      class="formatted font-config transition-opacity duration-500"
      ref="formatted"
      :style="{
        'line-height': store.lineHeight + 'px',
        'font-size': store.fontSize + 'px',
      }"
      :class="{
        'is-resizing': state.isResizingInnerPadding,
        'pointer-events-none': state.editMode === 'code',
        'opacity-0': !shiki,
      }"
    >
      <component v-once :is="() => shikiContent" />
    </div>

    <div
      class="font-config character-grid relative grid content-start"
      v-if="state.editMode === 'annotate'"
    >
      <template
        v-for="line in Array.from({
          length: block.content.split('\n').length,
        }).map((_, i) => i + 1)"
        :key="line"
      >
        <div
          class="flex"
          :style="{
            paddingLeft: gutter,
          }"
        >
          <div
            @click="addCharacterTransformer(line, character)"
            v-for="character in Array.from({ length: charactersPerLine }).map(
              (_, i) => i,
            )"
            :key="character"
            :style="{
              height: `${store.lineHeight}px`,
              width: `${characterWidth}px`,
            }"
            class="border border-transparent transition-colors hover:scale-x-110 hover:border-white/10 hover:bg-white/10"
          />
        </div>
      </template>
    </div>

    <textarea
      rows="1"
      class="editor font-config focus-visible:outline-none"
      ref="editor"
      v-model="block.content"
      spellcheck="false"
      data-enable-grammarly="false"
      :class="{
        'pointer-events-none': state.editMode !== 'code',
      }"
      :style="{
        'min-height':
          block.content.split('\n').length * store.lineHeight + 'px',
        'line-height': store.lineHeight + 'px',
        'font-size': store.fontSize + 'px',
      }"
    />
  </div>
</template>

<style>
.font-config {
  font-variation-settings: normal;
  font-feature-settings: v-bind(fontFeatureSettings);
  font-family: v-bind(fontFamily);
  font-size: 13px;
  -moz-tab-size: 2;
  -o-tab-size: 2;
  tab-size: 2;
  white-space-collapse: collapse;
  white-space: pre-wrap;
}

.editor,
.formatted,
.character-grid {
  grid-column-start: 1;
  grid-row-start: 1;
  height: 100%;
}

.editor {
  resize: none;
  background: transparent;
  border: none;
  margin-inline-end: v-bind(innerPaddingX);
  margin-inline-start: v-bind(gutter);
  -webkit-text-fill-color: transparent;
  -webkit-text-size-adjust: none;
  -moz-text-size-adjust: none;
  text-size-adjust: none;
  caret-color: white;
  outline: none;
  white-space: pre-wrap;
}

.formatted {
  user-select: none;
  position: relative;
}

.formatted code,
.formatted pre {
  font-family: inherit;
  font-feature-settings: inherit;
  font-variation-settings: inherit;
  background-color: transparent !important;
}

.formatted code {
  display: grid;
}

.formatted .has-focus .line:not(.focus) {
  opacity: 0.5;
  filter: blur(0.8px);
}

.formatted .line {
  position: relative;
  padding-inline-end: v-bind(innerPaddingX);
  padding-inline-start: v-bind(gutter);
  transition:
    padding 0.375s ease-in-out,
    opacity 0.375s ease-in-out,
    filter 0.075s ease-in-out;
}

.formatted.is-resizing .line {
  transition: none;
}

.formatted .line:hover {
  --highlight-opacity: 0.2;
  background: rgba(255, 255, 255, 0.1);
  cursor: pointer;
}

.is-light-theme .formatted .line:hover {
  background: hsla(200, 0%, 85%, 0.4);
}

.formatted .line-number {
  float: left;
  margin-left: calc(v-bind(gutter) * -1 - 4px);
  width: v-bind(gutter);
  padding-right: 0.5em;
  text-align: right;
  opacity: 0;
  user-select: none;
  transition:
    opacity 0.375s cubic-bezier(0.6, 0.6, 0, 1),
    0.375s cubic-bezier(0.6, 0.6, 0, 1);
}

.show-line-numbers .line-number {
  opacity: 1;
}

.formatted .line-number {
  color: rgba(255, 255, 255, 0.25);
}

.is-light-theme .formatted .line-number {
  color: rgba(0, 0, 0, 0.35);
}

.formatted .line span {
  white-space: pre-wrap;
}

.formatted .highlight {
  background: rgba(255, 255, 255, var(--highlight-opacity, 0.1));
}

.is-light-theme .formatted .highlight {
  background: hsla(200, 65%, 85%, var(--highlight-opacity, 0.7));
}

.formatted .highlight .line-number,
.formatted .add .line-number,
.formatted .remove .line-number {
  color: rgba(255, 255, 255, 0.6);
}

.is-light-theme .formatted .highlight .line-number,
.is-light-theme .formatted .add .line-number,
.is-light-theme .formatted .remove .line-number {
  color: rgba(0, 0, 0, 0.75);
}

.formatted .highlight .line-number {
  background: rgba(255, 255, 255, 0.1);
}

.is-light-theme .formatted .highlight .line-number {
  background: hsla(200, 70%, 45%, 0.15);
}

.formatted .line.add {
  background: rgba(0, 255, 0, var(--highlight-opacity, 0.15));
}

.is-light-theme .formatted .line.add {
  background: hsla(120, 70%, 45%, 0.2);
}

.formatted .add .line-number {
  background: rgba(0, 255, 0, var(--highlight-opacity, 0.1));
}

.is-light-theme .formatted .add .line-number {
  background: hsla(120, 70%, 45%, 0.15);
}

.formatted .line.remove {
  background: rgba(255, 0, 0, var(--highlight-opacity, 0.15));
}

.is-light-theme .formatted .line.remove {
  background: hsla(0, 70%, 45%, 0.2);
}

.formatted .remove .line-number {
  background: rgba(255, 0, 0, var(--highlight-opacity, 0.1));
}

.is-light-theme .formatted .remove .line-number {
  background: hsla(0, 70%, 45%, 0.15);
}

.annotation {
  margin-left: v-bind(gutter);
  position: relative;
  z-index: 100;
  margin-right: 20px;
}
</style>
