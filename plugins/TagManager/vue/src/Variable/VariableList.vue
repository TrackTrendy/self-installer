<!--
  Matomo - free/libre analytics platform
  @link https://matomo.org
  @license http://www.gnu.org/licenses/gpl-3.0.html GPL v3 or later
-->

<template>
  <div class="tagManagerManageList tagManagerVariableList">
    <ContentBlock
      feature="Tag Manager"
      class="tagManagerCustomVariablesList"
      :content-title="translate('TagManager_ManageX', translate('TagManager_Variables'))"
      :help-text="variablesHelpText"
    >
      <p>{{ translate('TagManager_VariableUsageBenefits') }}</p>
      <div class="variableSearchFilter">
        <Field
          uicontrol="text"
          name="variableSearch"
          :title="translate('General_Search')"
          v-show="variables.length > 0"
          v-model="variableSearch"
        >
        </Field>
      </div>
      <table v-content-table>
        <thead>
          <tr>
            <th class="name" :title="nameTranslatedText">{{ translate('General_Name') }}</th>
            <th class="description" :title="descriptionTranslatedText">
              {{ translate('General_Description') }}</th>
            <th class="type" :title="typeTranslatedText">{{ translate('TagManager_Type') }}</th>
            <th class="lookupTable"
              :title="lookupTableTranslatedText">{{ translate('TagManager_LookupTable') }}</th>
            <th class="lastUpdated"
              :title="lastUpdatedTranslatedText">{{ translate('TagManager_LastUpdated') }}</th>
            <th
              class="action"
              :title="actionTranslatedText"
              v-show="hasWriteAccess"
            >{{ translate('General_Actions') }}</th>
          </tr>
        </thead>
        <tbody>
          <tr v-show="isLoading || isUpdating">
            <td colspan="7">
              <span class="loadingPiwik">
                <img src="plugins/Morpheus/images/loading-blue.gif" />
                {{ translate('General_LoadingData') }}
              </span>
            </td>
          </tr>
          <tr v-show="!isLoading && !variables.length">
            <td colspan="7">
              {{ translate('TagManager_NoVariablesFound') }}
              <a
                class="createContainerVariableNow"
                v-show="hasWriteAccess"
                @click="createVariable()"
              >
                {{ translate('TagManager_CreateNewVariableNow') }}
              </a>
            </td>
          </tr>
          <tr
            :id="`variable${variable.idvariable}`"
            class="variables"
            v-for="variable in sortedVariables"
            :key="variable.idvariable"
          >
            <td class="name" :title="variable.name">{{ truncateText(variable.name, 50) }}</td>
            <td
              class="description"
              :title="variable.description"
            >
              {{ truncateText(variable.description, 75) }}
            </td>
            <td
              class="type"
              :title="variable.typeMetadata.description"
            >
              {{ variable.typeMetadata.name }}
            </td>
            <td class="lookupTable">
              <span
                class="icon-ok"
                v-show="variable.lookup_table.length"
              />
            </td>
            <td
              class="lastUpdated"
              :title="translate('TagManager_CreatedOnX', variable.created_date_pretty)"
            >
              <span>{{ variable.updated_date_pretty }}</span>
            </td>
            <td
              :class="getActionClasses"
              v-show="hasWriteAccess"
            >
              <a
                class="table-action icon-edit"
                @click="editVariable(variable.idvariable, variable.type)"
                :title="translate('TagManager_EditVariable')"
              />
              <a
                class="table-action icon-delete"
                @click="deleteVariable(variable)"
                :title="translate('TagManager_DeleteX', translate('TagManager_Variable'))"
              />
              <a
                class="table-action icon-content-copy"
                v-show="hasPublishCapability()"
                @click="openCopyDialog(variable)"
                :title="translate(
                  'TagManager_CopyX',
                  translate('TagManager_Variable'),
                )"
              />
            </td>
          </tr>
        </tbody>
      </table>
      <div
        class="tableActionBar"
        v-show="hasWriteAccess"
      >
        <a
          class="createNewVariable"
          value
          @click="createVariable()"
        >
          <span class="icon-add">&nbsp;</span>{{ translate('TagManager_CreateNewVariable') }}
        </a>
      </div>
    </ContentBlock>
    <h2 :title="translate('TagManager_PreConfiguredInfoTitle')">
      {{ translate('TagManager_PreconfiguredVariables') }}
      <span class="icon-help preconfiguredVariablesHelp" />
    </h2>
    <div>
      <ul
        class="collection with-header"
        v-for="(variableCategory, index) in containerVariables"
        :key="index"
      >
        <li class="collection-header">
          <h4>{{ variableCategory.name }}</h4>
        </li>
        <li
          class="collection-item"
          v-show="variableTemplate.is_pre_configured"
          v-for="variableTemplate in variableCategory.types"
          :key="variableTemplate.id"
        >
          <span class="title">
            {{ variableTemplate.name }}
            <span class="variableId" v-text="`{{${variableTemplate.id}}}`"></span>
          </span>
          <span
            class="secondary-content"
            v-show="!!variableTemplate.description"
          >
            <i
              class="icon-help"
              :title="variableTemplate.description"
            />
          </span>
        </li>
      </ul>
    </div>
    <div
      class="ui-confirm"
      id="confirmDeleteVariable"
      ref="confirmDeleteVariable"
    >
      <h2>{{ translate('TagManager_DeleteVariableConfirm') }} </h2>
      <input
        role="yes"
        type="button"
        :value="translate('General_Yes')"
      />
      <input
        role="no"
        type="button"
        :value="translate('General_No')"
      />
    </div>
    <div
      class="ui-confirm"
      id="confirmDeleteVariableNotPossible"
      ref="confirmDeleteVariableNotPossible"
    >
      <h2>{{ translate('TagManager_VariableCannotBeDeleted') }}</h2>
      <p>
        {{ translate('TagManager_VariableBeingUsedBy') }}
      </p>
      <ul class="collection">
        <li
          class="collection-item"
          v-for="reference in variableReferences"
          :key="`${reference.referenceType}.${reference.referenceId}`"
        >
          {{ reference.referenceTypeName }}: {{ reference.referenceName }}
        </li>
      </ul>
      <p>
        {{ translate('TagManager_VariableBeingUsedNeedsRemove') }}
      </p>
      <input
        role="no"
        type="button"
        :value="translate('General_Cancel')"
      />
    </div>
  </div>
</template>

<script lang="ts">
import { DeepReadonly, defineComponent } from 'vue';
import {
  AjaxHelper,
  ContentBlock,
  ContentTable,
  Matomo,
  MatomoUrl,
  NotificationsStore,
} from 'CoreHome';
import { Field } from 'CorePluginsAdmin';
import VariablesStore from './Variables.store';
import {
  VariableReference,
  ContainerVariableCategory,
  Variable, VariableType,
} from '../types';

interface VariableListState {
  hasWriteAccess: boolean;
  variableReferences: VariableReference[];
  containerVariables: ContainerVariableCategory[];
  variableSearch: string;
}

const { tagManagerHelper } = window;

export default defineComponent({
  props: {
    idContainer: {
      type: String,
      required: true,
    },
    idContainerVersion: {
      type: Number,
      required: true,
    },
    variablesHelpText: String,
  },
  components: {
    Field,
    ContentBlock,
  },
  directives: {
    ContentTable,
  },
  data(): VariableListState {
    return {
      hasWriteAccess: Matomo.hasUserCapability('tagmanager_write'),
      variableReferences: [],
      containerVariables: [],
      variableSearch: '',
    };
  },
  created() {
    VariablesStore.fetchVariables(this.idContainer, this.idContainerVersion);

    AjaxHelper.fetch<ContainerVariableCategory[]>({
      method: 'TagManager.getAvailableContainerVariables',
      idContainer: this.idContainer,
      idContainerVersion: this.idContainerVersion,
      filter_limit: '-1',
    }).then((variables) => {
      this.containerVariables = variables;
    });
  },
  methods: {
    createVariable() {
      this.editVariable(0);
    },
    editVariable(idVariable: number) {
      MatomoUrl.updateHash({
        ...MatomoUrl.hashParsed.value,
        idVariable,
      });
    },
    deleteVariable(variable: DeepReadonly<Variable>) {
      AjaxHelper.fetch<VariableReference[]>({
        method: 'TagManager.getContainerVariableReferences',
        idContainer: this.idContainer,
        idContainerVersion: this.idContainerVersion,
        idVariable: variable.idvariable,
      }).then((references) => {
        if (!references || !references.length) {
          this.variableReferences = [];

          Matomo.helper.modalConfirm(this.$refs.confirmDeleteVariable as HTMLElement, {
            yes: () => {
              VariablesStore.deleteVariable(
                this.idContainer,
                this.idContainerVersion,
                variable.idvariable!,
              ).then(() => {
                VariablesStore.reload(this.idContainer, this.idContainerVersion);
                NotificationsStore.remove('CopyDialogResultNotification');
              });
            },
          });
        } else {
          this.variableReferences = references;
          Matomo.helper.modalConfirm(
            this.$refs.confirmDeleteVariableNotPossible as HTMLElement,
            {},
          );
        }
      });
    },
    truncateText(text: string, length: number) {
      return tagManagerHelper.truncateText(text, length);
    },
    hasPublishCapability() {
      return Matomo.hasUserCapability('tagmanager_write') && Matomo.hasUserCapability('tagmanager_use_custom_templates');
    },
    openCopyDialog(variable: Variable) {
      const url = MatomoUrl.stringify({
        module: 'TagManager',
        action: 'copyVariableDialog',
        idSite: variable.idsite,
        idContainer: this.idContainer,
        idVariable: variable.idvariable,
        idContainerVersion: this.idContainerVersion,
      });
      window.Piwik_Popover.createPopupAndLoadUrl(url, '', 'mtmCopyVariable');
    },
  },
  computed: {
    isLoading() {
      return VariablesStore.isLoading.value;
    },
    isUpdating() {
      return VariablesStore.isUpdating.value;
    },
    variables() {
      return VariablesStore.variables.value;
    },
    sortedVariables() {
      const searchFilter = this.variableSearch.toLowerCase();

      // look through string properties of variables for values that have searchFilter in them
      // (mimics angularjs filter() filter)
      const result = [...this.variables].filter((h) => Object.keys(h).some((propName) => {
        const entity = h as unknown as Record<string, unknown>;
        let propValue = '';
        if (typeof entity[propName] === 'string') {
          propValue = (entity[propName] as string);
        } else if (propName === 'typeMetadata') {
          const propTypeMeta = (entity.typeMetadata as VariableType);
          propValue = (propTypeMeta.name as string);
        } else if (propName === 'parameters' && entity.type === 'CustomJsFunction') {
          const propTypeParameters = (entity.parameters as Record<string, unknown>);
          propValue = (propTypeParameters.jsFunction as string);
        }
        return propValue.toLowerCase().indexOf(searchFilter) !== -1;
      }));
      result.sort((lhs, rhs) => {
        if (lhs.name < rhs.name) {
          return -1;
        }
        return lhs.name > rhs.name ? 1 : 0;
      });
      return result;
    },
    nameTranslatedText(): string {
      return this.translate('TagManager_VariablesNameDescription');
    },
    descriptionTranslatedText(): string {
      return this.translate('TagManager_VariablesDescriptionDescription');
    },
    typeTranslatedText(): string {
      return this.translate('TagManager_VariablesTypeDescription');
    },
    lookupTableTranslatedText(): string {
      return this.translate('TagManager_VariablesLookupTableDescription');
    },
    lastUpdatedTranslatedText(): string {
      return this.translate('TagManager_VariablesLastUpdatedDescription');
    },
    actionTranslatedText(): string {
      return this.translate('TagManager_VariablesActionDescription');
    },
    getActionClasses(): string {
      const copyClass = this.hasPublishCapability() ? ' hasCopyAction' : '';
      return `action${copyClass}`;
    },
  },
});
</script>
