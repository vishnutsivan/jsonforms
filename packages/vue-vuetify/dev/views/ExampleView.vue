<script setup lang="ts">
import type { JsonFormsChangeEvent } from '@jsonforms/vue';
import type { ErrorObject } from 'ajv';
import cloneDeep from 'lodash/cloneDeep';
import find from 'lodash/find';
import * as monaco from 'monaco-editor/esm/vs/editor/editor.api';
import {
  markRaw,
  onMounted,
  provide,
  ref,
  shallowReactive,
  shallowRef,
  watch,
  type ShallowRef,
} from 'vue';
import type { ExampleDescription } from '../../../examples/lib';
import { ValidationIcon, defaultStyles, mergeStyles } from '../../src';
import ExampleForm from '../components/ExampleForm.vue';
import MonacoEditor from '../components/MonacoEditor.vue';
import {
  configureDataValidation,
  configureJsonSchemaValidation,
  configureUISchemaValidation,
  getMonacoModelForUri,
} from '../core/jsonSchemaValidation';
import type { MonacoApi } from '../core/monaco';
import examples from '../examples';
import { useAppStore } from '../store';
import { createAjv } from '../validate';

import { Pane, Splitpanes } from 'splitpanes';
import 'splitpanes/dist/splitpanes.css';

const { extendedVuetifyRenderers } = await import('../../src');

const props = defineProps<{
  example: ExampleDescription;
}>();

const appStore = useAppStore();

const myStyles = mergeStyles(defaultStyles, {
  control: { root: 'my-control' },
});

provide('styles', myStyles);

const ajv = createAjv();
const errors = ref<
  ErrorObject<string, Record<string, any>, unknown>[] | undefined
>(undefined);
const snackbar = ref(false);
const snackbarText = ref('');
const snackbarTimeout = ref(3000);

const renderers = markRaw(extendedVuetifyRenderers);

const schemaModel = shallowRef<monaco.editor.ITextModel | undefined>(undefined);
const uischemaModel = shallowRef<monaco.editor.ITextModel | undefined>(
  undefined,
);
const dataModel = shallowRef<monaco.editor.ITextModel | undefined>(undefined);

const initialState = (exampleProp: ExampleDescription) => {
  const example = cloneDeep(exampleProp);

  return {
    data: example.data,
    schema: example.schema,
    uischema: example.uischema,
    renderers: renderers,
    cells: undefined, // not defined
    config: appStore.jsonforms.config,
    readonly: appStore.jsonforms.readonly,
    uischemas: example.uischemas,
    validationMode: appStore.jsonforms.validationMode,
    ajv: ajv,
    i18n: exampleProp.i18n ?? {
      locale: appStore.jsonforms.locale,
    },
    additionalErrors: undefined,
    middleware: undefined,
  };
};
const state = shallowReactive(initialState(props.example));

const onChange = (event: JsonFormsChangeEvent): void => {
  if (props.example.name) {
    dataModel.value = getMonacoModelForUri(
      monaco.Uri.parse(toDataUri(props.example.name)),
      event.data !== undefined ? JSON.stringify(event.data, null, 2) : '',
    );
    state.data = event.data;
  }
  errors.value = event.errors;
};

const reloadMonacoSchema = () => {
  const example = find(
    examples,
    (example) => example.name === appStore.exampleName,
  );

  if (example) {
    schemaModel.value = getMonacoModelForUri(
      monaco.Uri.parse(toSchemaUri(example.name)),
      example.schema ? JSON.stringify(example.schema, null, 2) : '',
    );
    toast('Original example schema loaded. Apply it to take effect.');
  }
};

const saveMonacoSchema = () => {
  saveMonacoModel(
    schemaModel,
    (modelValue) => (state.schema = JSON.parse(modelValue)),
    'New schema applied',
  );

  if (state.schema) {
    configureDataValidation(
      monaco,
      `inmemory://${toSchemaUri(props.example.name)}`,
      toDataUri(props.example.name),
      cloneDeep(state.schema),
    );
  }
};

const reloadMonacoUiSchema = () => {
  const example = find(
    examples,
    (example) => example.name === appStore.exampleName,
  );

  if (example) {
    uischemaModel.value = getMonacoModelForUri(
      monaco.Uri.parse(toUiSchemaUri(example.name)),
      example.uischema ? JSON.stringify(example.uischema, null, 2) : '',
    );
    toast('Original example UI schema loaded. Apply it to take effect.');
  }
};

const saveMonacoUiSchema = () => {
  saveMonacoModel(
    uischemaModel,
    (modelValue) =>
      (state.uischema = modelValue ? JSON.parse(modelValue) : undefined),
    'New UI schema applied',
  );
};

const reloadMonacoData = () => {
  const example = find(
    examples,
    (example) => example.name === appStore.exampleName,
  );

  if (example) {
    dataModel.value = getMonacoModelForUri(
      monaco.Uri.parse(toDataUri(example.name)),
      example.data !== undefined ? JSON.stringify(example.data, null, 2) : '',
    );
    toast('Original example data loaded. Apply it to take effect.');
  }
};

const saveMonacoData = () => {
  saveMonacoModel(
    dataModel,
    (modelValue) => {
      state.data = modelValue === '' ? undefined : JSON.parse(modelValue);
    },
    'New data applied',
  );
};

const saveMonacoModel = (
  model: ShallowRef<monaco.editor.ITextModel | undefined>,
  apply: (value: string) => void,
  successToast: string,
) => {
  if (model.value) {
    const modelValue = model.value.getValue();

    try {
      apply(modelValue);
      toast(successToast);
    } catch (error) {
      toast(`Error: ${error}`);
    }
  }
};

const registerValidations = (editor: MonacoApi) => {
  configureJsonSchemaValidation(editor, ['*.schema.json']);
  configureUISchemaValidation(editor, ['*.uischema.json']);
  for (const example of examples) {
    const schema = {
      ...example.schema,
      title: example.label,
    };

    if (example && example.schema) {
      configureDataValidation(
        editor,
        `inmemory://${toSchemaUri(example.name)}`,
        toDataUri(example.name),
        schema,
      );
    }
  }
};

const updateMonacoModels = (example: ExampleDescription) => {
  schemaModel.value = getMonacoModelForUri(
    monaco.Uri.parse(toSchemaUri(example.name)),
    example.schema ? JSON.stringify(example.schema, null, 2) : '',
  );

  uischemaModel.value = getMonacoModelForUri(
    monaco.Uri.parse(toUiSchemaUri(example.name)),
    example.uischema ? JSON.stringify(example.uischema, null, 2) : '',
  );

  dataModel.value = getMonacoModelForUri(
    monaco.Uri.parse(toDataUri(example.name)),
    example.data !== undefined ? JSON.stringify(example.data, null, 2) : '',
  );
};

const toSchemaUri = (id: string): string => {
  return `${id}.schema.json`;
};
const toUiSchemaUri = (id: string): string => {
  return `${id}.uischema.json`;
};
const toDataUri = (id: string): string => {
  return `${id}.data.json`;
};
const toI18NUri = (id: string): string => {
  return `${id}.i18n.json`;
};
const toast = (message: string): void => {
  snackbar.value = true;
  snackbarText.value = message;
};

onMounted(() => {
  updateMonacoModels(props.example);
});

watch(
  () => props.example,
  (value) => {
    updateMonacoModels(props.example);
    // reset state when example changes
    Object.assign(state, initialState(value));
  },
);

watch(
  () => appStore.formOnly,
  (value) => {
    if (!value) {
      // we need to show the wrapper so make sure that the monaco models are updated
      updateMonacoModels(props.example);
    }
  },
);

type Action = NonNullable<ExampleDescription['actions']>[number];

const handleAction = (action: Action) => {
  if (action) {
    const newState = action.apply(state);
    if (newState) {
      Object.assign(state, newState);
    }
  }
};
</script>

<template>
  <div>
    <v-container fluid class="demo" v-if="!appStore.formOnly">
      <v-card>
        <v-card-title>{{ example.label }}</v-card-title>
        <v-card-text>
          <v-tabs v-model="appStore.activeTab">
            <v-tab :key="0"
              >{{ appStore.layout == 'demo-and-data' ? 'Demo and Data' : 'Demo'
              }}<validation-icon
                v-if="errors"
                :errors="errors"
              ></validation-icon
            ></v-tab>
            <v-spacer expand />
            <v-tab :key="1">Schema</v-tab>
            <v-tab :key="2">UI Schema</v-tab>
            <v-tab v-if="appStore.layout !== 'demo-and-data'" :key="3"
              >Data</v-tab
            >
          </v-tabs>
        </v-card-text>
        <v-window v-model="appStore.activeTab">
          <v-window-item :key="0">
            <v-card>
              <v-card-title>
                <v-toolbar flat>
                  <v-toolbar-title>JSONForm</v-toolbar-title>
                  <v-spacer></v-spacer>
                  <v-toolbar-items v-if="example.actions">
                    <v-btn
                      v-for="(action, index) in example.actions"
                      :key="index"
                      @click="() => handleAction(action)"
                      >{{ action.label }}</v-btn
                    >
                  </v-toolbar-items>
                </v-toolbar>
              </v-card-title>
              <v-divider class="mx-4"></v-divider>
              <div class="json-forms">
                <splitpanes
                  :class="['default-theme', 'splitpanes-vuetify']"
                  :rtl="appStore.rtl"
                  v-if="appStore.layout === 'demo-and-data'"
                >
                  <pane min-size="20">
                    <v-card>
                      <v-card-title>
                        <v-toolbar flat>
                          <v-toolbar-title>Demo</v-toolbar-title>
                          <v-spacer></v-spacer>
                        </v-toolbar>
                      </v-card-title>
                      <v-divider class="mx-4"></v-divider>
                      <example-form :state="state" @jsfchange="onChange" />
                    </v-card>
                  </pane>
                  <pane>
                    <v-card>
                      <v-card-title>
                        <v-toolbar flat>
                          <v-toolbar-title>Data</v-toolbar-title>
                          <v-spacer></v-spacer>
                          <v-tooltip bottom>
                            <template v-slot:activator="{ props }">
                              <v-btn
                                icon
                                @click="reloadMonacoData"
                                v-bind="props"
                              >
                                <v-icon>$reload</v-icon>
                              </v-btn>
                            </template>
                            {{ `Reload Example Data` }}
                          </v-tooltip>
                          <v-tooltip bottom>
                            <template v-slot:activator="{ props }">
                              <v-btn
                                icon
                                @click="saveMonacoData"
                                v-bind="props"
                              >
                                <v-icon>$save</v-icon>
                              </v-btn>
                            </template>
                            {{ `Apply Change To Example Data` }}
                          </v-tooltip>
                        </v-toolbar>
                      </v-card-title>
                      <v-divider class="mx-4"></v-divider>
                      <monaco-editor
                        language="json"
                        v-model="dataModel"
                        style="height: calc(100vh - 200px)"
                        :editorBeforeMount="registerValidations"
                      ></monaco-editor>
                    </v-card>
                  </pane>
                </splitpanes>

                <example-form :state="state" @jsfchange="onChange" v-else />
              </div>
            </v-card>
          </v-window-item>
          <v-window-item :key="1">
            <v-card>
              <v-card-title>
                <v-toolbar flat>
                  <v-toolbar-title>Schema</v-toolbar-title>
                  <v-spacer></v-spacer>
                  <v-tooltip bottom>
                    <template v-slot:activator="{ props }">
                      <v-btn icon @click="reloadMonacoSchema" v-bind="props">
                        <v-icon>$reload</v-icon>
                      </v-btn>
                    </template>
                    {{ `Reload Example Schema` }}
                  </v-tooltip>
                  <v-tooltip bottom>
                    <template v-slot:activator="{ props }">
                      <v-btn icon @click="saveMonacoSchema" v-bind="props">
                        <v-icon>$save</v-icon>
                      </v-btn>
                    </template>
                    {{ `Apply Change To Example Schema` }}
                  </v-tooltip>
                </v-toolbar>
              </v-card-title>
              <v-divider class="mx-4"></v-divider>
              <monaco-editor
                :language="`json`"
                v-model="schemaModel"
                style="height: calc(100vh - 100px)"
                :editorBeforeMount="registerValidations"
              ></monaco-editor>
            </v-card>
          </v-window-item>
          <v-window-item :key="2">
            <v-card>
              <v-card-title>
                <v-toolbar flat>
                  <v-toolbar-title>UI Schema</v-toolbar-title>
                  <v-spacer></v-spacer>
                  <v-tooltip bottom>
                    <template v-slot:activator="{ props }">
                      <v-btn icon @click="reloadMonacoUiSchema" v-bind="props">
                        <v-icon>$reload</v-icon>
                      </v-btn>
                    </template>
                    {{ `Reload Example UI Schema` }}
                  </v-tooltip>
                  <v-tooltip bottom>
                    <template v-slot:activator="{ props }">
                      <v-btn icon @click="saveMonacoUiSchema" v-bind="props">
                        <v-icon>$save</v-icon>
                      </v-btn>
                    </template>
                    {{ `Apply Change To Example UI Schema` }}
                  </v-tooltip>
                </v-toolbar>
              </v-card-title>
              <v-divider class="mx-4"></v-divider>
              <monaco-editor
                language="json"
                v-model="uischemaModel"
                style="height: calc(100vh - 100px)"
                :editorBeforeMount="registerValidations"
              ></monaco-editor>
            </v-card>
          </v-window-item>
          <v-window-item :key="3">
            <v-card>
              <v-card-title>
                <v-toolbar flat>
                  <v-toolbar-title>Data</v-toolbar-title>
                  <v-spacer></v-spacer>
                  <v-tooltip bottom>
                    <template v-slot:activator="{ props }">
                      <v-btn icon @click="reloadMonacoData" v-bind="props">
                        <v-icon>$reload</v-icon>
                      </v-btn>
                    </template>
                    {{ `Reload Example Data` }}
                  </v-tooltip>
                  <v-tooltip bottom>
                    <template v-slot:activator="{ props }">
                      <v-btn icon @click="saveMonacoData" v-bind="props">
                        <v-icon>$save</v-icon>
                      </v-btn>
                    </template>
                    {{ `Apply Change To Example Data` }}
                  </v-tooltip>
                </v-toolbar>
              </v-card-title>
              <v-divider class="mx-4"></v-divider>
              <monaco-editor
                language="json"
                v-model="dataModel"
                style="height: calc(100vh - 100px)"
                :editorBeforeMount="registerValidations"
              ></monaco-editor>
            </v-card>
          </v-window-item>
        </v-window>
      </v-card>
      <v-snackbar v-model="snackbar" :timeout="snackbarTimeout">
        {{ snackbarText }}
        <template v-slot:actions>
          <v-btn variant="text" @click="snackbar = false"> Close </v-btn>
        </template>
      </v-snackbar>
    </v-container>
    <div class="json-forms" v-else>
      <example-form :state="state" @jsfchange="onChange" />
    </div>
  </div>
</template>
<style lang="scss" scoped>
:deep(.default-theme) {
  &.splitpanes--vertical > .splitpanes__splitter,
  .splitpanes--vertical > .splitpanes__splitter {
    border-left: 1px solid rgb(var(--v-theme-on-surface-variant));
  }

  &.splitpanes--horizontal > .splitpanes__splitter,
  .splitpanes--horizontal > .splitpanes__splitter {
    border-top: 1px solid rgb(var(--v-theme-on-surface-variant));
  }

  .splitpanes__splitter {
    background-color: rgb(var(--v-theme-surface));
    &:before,
    &:after {
      background-color: rgb(var(--v-theme-on-surface-variant));
    }
    &:hover:before,
    &:hover:after {
      background-color: rgb(var(--v-theme-on-surface-variant));
    }
  }
}
</style>
