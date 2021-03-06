/**
 * src/components/sys-form.vue
 * This is a super smart form that handles validation, error checking, and
 * async submitting.
 */

<template>
  <validation-observer
    ref="observer"
    v-slot="binds"
    :class="$style.form"
    :disabled="disabled"
    tag="form"
    @submit.prevent="submit"
  >
    <template v-if="error">
      <slot
        name="error"
        v-bind="{ error }"
      >
        <sys-form-error>
          {{ error.message }}
        </sys-form-error>
      </slot>
    </template>

    <slot v-bind="{ submitting, ...binds }" />

    <div :class="actionsClasses">
      <slot
        name="actions"
        v-bind="{ submitting, submittable: (binds.valid && !submitting), ...binds }"
      >
        <sys-form-button
          :color="submitColor"
          :disabled="!binds.valid || submitting"
        >
          {{ submitText }}
        </sys-form-button>
      </slot>
    </div>
  </validation-observer>
</template>

<script>
  import { ValidationObserver } from 'vee-validate'
  import Vue from 'vue'

  import SysFormButton from './sys-form-button.vue'
  import SysFormError from './sys-form-error.vue'

  export default {
    name: 'SysForm',

    components: {
      SysFormButton,
      SysFormError,
      ValidationObserver
    },

    inheritAttrs: false,

    props: {
      /** If the whole form should be disabled */
      disabled: {
        type: Boolean,
        default: false
      },

      /**
       * Invalid text to to display on the form. The form will still be
       * submittable with this text.
       */
      invalid: {
        type: [String, Object],
        default: null
      },

      /** The color the submit button should be */
      submitColor: {
        type: String,
        default: 'primary',
        validator: (v) => ['normal', 'primary', 'secondary'].includes(v)
      },

      /** The alignment of the form buttons */
      submitAlign: {
        type: String,
        default: 'right',
        validator: (v) => ['left', 'right'].includes(v)
      },

      /**
       * The function to run when the user submits. This will only occur when
       * validation passes.
       */
      submitFunction: {
        type: Function,
        default: () => null
      },

      /** The text the submit buton should have */
      submitText: {
        type: String,
        default: 'Submit'
      },

      /**
       * If we should pass any error that occured in the form to Vue. This will
       * usually get to your third party error library, but may also change the
       * page to a 500 error page.
       */
      swallowError: {
        type: Boolean,
        default: true
      }
    },

    data: () => ({
      formError: null,
      submitting: false
    }),

    computed: {
      actionsClasses () {
        return {
          [this.$style.actions]: true,
          [this.$style.left]: (this.submitAlign === 'left'),
          [this.$style.right]: (this.submitAlign === 'right')
        }
      },

      error () {
        if (typeof this.invalid === 'string') {
          return new Error(this.invalid)
        } else if (this.invalid != null) {
          return this.invalid
        } else {
          return this.formError
        }
      }
    },

    errorCaptured (err, vm, info) {
      this.handleError(err, vm, info)
    },

    methods: {
      handleError (err, vm, info) {
        console.error(err)
        this.formError = err

        if (!this.swallowError && Vue.config != null && Vue.config.errorHandler != null) {
          Vue.config.errorHandler(err, vm || this, info)
        }
      },

      async submit (e) {
        if (this.disabled) {
          e.preventDefault()
          return false
        }

        this.submitting = true

        try {
          const valid = await this.$refs.observer.validate()

          if (!valid) {
            e.preventDefault()
            return false
          } else {
            await this.submitFunction()
          }
        } catch (err) {
          this.handleError(err)
        } finally {
          this.submitting = false
        }
      }
    }
  }
</script>

<style module>
  .form {
    display: block;
    margin: 0;
    min-width: calc(320px - 2rem);
    padding: 0;
  }

  .actions {
    align-content: center;
    align-items: center;
    display: flex;
    flex-wrap: wrap;
    margin: -0.4rem 0 0;
  }

  .left {
    justify-content: flex-start;
  }

  .right {
    justify-content: flex-end;
  }
</style>
