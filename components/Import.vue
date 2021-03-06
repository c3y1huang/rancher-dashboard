<script>
import { mapGetters } from 'vuex';
import Card from '@/components/Card';
import Banner from '@/components/Banner';
import Loading from '@/components/Loading';
import YamlEditor from '@/components/YamlEditor';
import FileSelector from '@/components/form/FileSelector';
import AsyncButton from '@/components/AsyncButton';
import LabeledSelect from '@/components/form/LabeledSelect';
import SortableTable from '@/components/SortableTable';
import { sortBy } from '@/utils/sort';
import { exceptionToErrorsArray } from '@/utils/error';
import { NAMESPACE } from '@/config/types';
import {
  NAME as NAME_COL, STATE, TYPE, NAMESPACE as NAMESPACE_COL, AGE
} from '@/config/table-headers';

export default {
  components: {
    AsyncButton,
    Banner,
    Card,
    Loading,
    YamlEditor,
    FileSelector,
    LabeledSelect,
    SortableTable
  },

  async fetch() {
    this.allNamespaces = await this.$store.dispatch('cluster/findAll', { type: NAMESPACE, opt: { url: 'namespaces' } });
  },

  data() {
    return {
      currentYaml:      '',
      defaultNamespace: 'default',
      allNamespaces:    null,
      errors:           null,
      rows:             null,
      done:             false,
    };
  },

  computed: {
    ...mapGetters(['currentCluster']),

    namespaceOptions() {
      const out = this.allNamespaces.map((obj) => {
        return {
          label: obj.name,
          value: obj.value,
        };
      });

      return sortBy(out, 'label');
    },

    headers() {
      return [
        STATE,
        TYPE,
        NAME_COL,
        NAMESPACE_COL,
        AGE
      ];
    },
  },

  methods: {
    close() {
      this.$emit('close');
    },

    onFileSelected(value) {
      // const component = this.$refs.yamleditor;

      // if (component) {
      // component.updateValue(value);
      // }

      this.currentYaml = value;
    },

    async importYaml(btnCb) {
      try {
        const res = await this.currentCluster.doAction('apply', {
          yaml:             this.currentYaml,
          defaultNamespace: this.defaultNamespace,
        });

        btnCb(true);

        this.rows = res;
        this.done = true;
      } catch (err) {
        this.errors = exceptionToErrorsArray(err);
        this.done = false;
        btnCb(false);
      }
    },
  },
};
</script>

<template>
  <Loading v-if="$fetchState.pending" />
  <Card v-else :show-highlight-border="false">
    <template #title>
      <div style="display: block; width: 100%;">
        <template v-if="done">
          <h4>{{ t('import.success', {count: rows.length}) }}</h4>
        </template>
        <template v-else>
          <h4 v-t="'import.title'"></h4>
          <div class="row">
            <div class="col span-6">
              <FileSelector
                class="btn role-secondary pull-left"
                :label="t('generic.readFromFile')"
                @selected="onFileSelected"
              />
            </div>
            <div class="col span-6">
              <LabeledSelect
                v-model="defaultNamespace"
                class="pull-right"
                :options="namespaceOptions"
                label-key="import.defaultNamespace.label"
                mode="edit"
              />
            </div>
          </div>
        </template>
      </div>
    </template>
    <template #body>
      <template v-if="done">
        <Banner color="success" :label="t('import.success', {count: rows.length})" />
        <div class="results">
          <SortableTable
            :rows="rows"
            :headers="headers"
            mode="view"
            key-field="_key"
            :search="false"
            :paging="true"
            :row-actions="false"
            :table-actions="false"
          />
        </div>
      </template>
      <YamlEditor
        v-else
        ref="yamleditor"
        v-model="currentYaml"
        class="yaml-editor"
      />
    </template>
    <template #actions>
      <Banner v-for="(err, i) in errors" :key="i" color="error" :label="err" />
      <div v-if="done" class="text-center" style="width: 100%">
        <button type="button" class="btn role-primary" @click="close">
          {{ t('generic.close') }}
        </button>
      </div>
      <div v-else class="text-center" style="width: 100%">
        <button type="button" class="btn role-secondary" @click="close">
          {{ t('generic.cancel') }}
        </button>
        <AsyncButton v-if="!done" mode="import" :disabled="!currentYaml.length" @click="importYaml" />
      </div>
    </template>
  </Card>
</template>

<style lang='scss' scoped>
  $min: 50vh;
  $max: 50vh;

  .yaml-editor {
    flex: 1;
    min-height: $min;
    max-height: $max;

    ::v-deep .code-mirror {
      .CodeMirror {
        position: initial;
      }

      .CodeMirror,
      .CodeMirror-scroll,
      .CodeMirror-gutters {
        min-height: $min;
        max-height: $max;
      }
    }
  }
</style>
