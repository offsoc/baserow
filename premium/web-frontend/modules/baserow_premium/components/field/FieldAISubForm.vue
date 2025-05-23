<template>
  <div v-if="!isDeactivated" class="context__form-container">
    <SelectAIModelForm
      :default-values="defaultValues"
      :database="database"
      @ai-type-changed="setFileFieldSupported"
    ></SelectAIModelForm>

    <FormGroup v-if="fileFieldSupported" required small-label>
      <template #label>
        {{ $t('selectAIModelForm.fileField') }}
        <HelpIcon
          :tooltip="$t('fieldAISubForm.fileFieldHelp')"
          :tooltip-content-classes="['tooltip__content--expandable']"
        />
      </template>

      <Dropdown
        v-model="v$.values.ai_file_field_id.$model"
        class="dropdown--floating"
        :error="fieldHasErrors('ai_file_field_id')"
        :fixed-items="true"
        :show-search="false"
        @hide="v$.values.ai_file_field_id.$touch"
      >
        <DropdownItem
          :name="$t('fieldAISubForm.emptyFileField')"
          :value="null"
        />
        <DropdownItem
          v-for="field in fileFields"
          :key="field.id"
          :name="field.name"
          :value="field.id"
        />
      </Dropdown>
    </FormGroup>

    <FormGroup
      required
      small-label
      :label="$t('fieldAISubForm.outputType')"
      :help-icon-tooltip="$t('fieldAISubForm.outputTypeTooltip')"
    >
      <Dropdown
        v-model="v$.values.ai_output_type.$model"
        class="dropdown--floating"
        :fixed-items="true"
      >
        <DropdownItem
          v-for="outputTypeItem in outputTypes"
          :key="outputTypeItem.getType()"
          :name="outputTypeItem.getName()"
          :value="outputTypeItem.getType()"
          :description="outputTypeItem.getDescription()"
        />
      </Dropdown>
      <template v-if="changedOutputType" #warning>
        {{ $t('fieldAISubForm.outputTypeChangedWarning') }}
      </template>
    </FormGroup>

    <FormGroup
      small-label
      :label="$t('fieldAISubForm.prompt')"
      :error="fieldHasErrors('ai_prompt')"
      required
    >
      <div style="max-width: 366px">
        <FormulaInputField
          v-model="v$.values.ai_prompt.$model"
          :data-providers="dataProviders"
          :application-context="applicationContext"
          :placeholder="$t('fieldAISubForm.promptPlaceholder')"
          @input="v$.values.ai_prompt.$touch()"
        ></FormulaInputField>
      </div>
      <template #error> {{ $t('error.requiredField') }}</template>
    </FormGroup>

    <component
      :is="outputType.getFormComponent()"
      ref="childForm"
      v-bind="$props"
    />
  </div>
  <div v-else>
    <p>
      {{ $t('fieldAISubForm.premiumFeature') }} <i class="iconoir-lock"></i>
    </p>
  </div>
</template>

<script>
import { useVuelidate } from '@vuelidate/core'
import { required } from '@vuelidate/validators'
import form from '@baserow/modules/core/mixins/form'
import fieldSubForm from '@baserow/modules/database/mixins/fieldSubForm'
import FormulaInputField from '@baserow/modules/core/components/formula/FormulaInputField'
import SelectAIModelForm from '@baserow/modules/core/components/ai/SelectAIModelForm'
import { TextAIFieldOutputType } from '@baserow_premium/aiFieldOutputTypes'

export default {
  name: 'FieldAISubForm',
  components: { SelectAIModelForm, FormulaInputField },
  mixins: [form, fieldSubForm],
  setup() {
    return { v$: useVuelidate({ $lazy: true }) }
  },
  data() {
    return {
      allowedValues: ['ai_prompt', 'ai_file_field_id', 'ai_output_type'],
      values: {
        ai_prompt: '',
        ai_output_type: TextAIFieldOutputType.getType(),
        ai_file_field_id: null,
      },
      fileFieldSupported: false,
    }
  },
  computed: {
    // Return the reactive object that can be updated in runtime.
    workspace() {
      return this.$store.getters['workspace/get'](this.database.workspace.id)
    },
    applicationContext() {
      const context = {}
      Object.defineProperty(context, 'fields', {
        enumerable: true,
        get: () =>
          this.allFieldsInTable.filter((f) => {
            const isNotThisField = f.id !== this.defaultValues.id
            return isNotThisField
          }),
      })
      return context
    },
    dataProviders() {
      return [this.$registry.get('databaseDataProvider', 'fields')]
    },
    isDeactivated() {
      return this.$registry
        .get('field', this.fieldType)
        .isDeactivated(this.workspace.id)
    },
    fileFields() {
      return this.allFieldsInTable.filter((field) => {
        const t = this.$registry.get('field', field.type)
        return t.canRepresentFiles(field)
      })
    },
    outputTypes() {
      return Object.values(this.$registry.getAll('aiFieldOutputType'))
    },
    outputType() {
      return this.$registry.get('aiFieldOutputType', this.values.ai_output_type)
    },
    changedOutputType() {
      return (
        this.defaultValues.id &&
        this.defaultValues.type === this.values.type &&
        this.defaultValues.ai_output_type !== this.values.ai_output_type
      )
    },
  },
  methods: {
    setFileFieldSupported(generativeAIType) {
      if (generativeAIType) {
        const modelType = this.$registry.get(
          'generativeAIModel',
          generativeAIType
        )
        this.fileFieldSupported = modelType.canPromptWithFiles()
      } else {
        this.fileFieldSupported = false
      }

      if (!this.fileFieldSupported) {
        this.values.ai_file_field_id = null
      }
    },
  },
  validations() {
    return {
      values: {
        ai_prompt: { required },
        ai_file_field_id: {},
        ai_output_type: {},
      },
    }
  },
}
</script>
