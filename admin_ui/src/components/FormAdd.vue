<template>
  <div>
    <template v-if="formConfig && !successMessage">
      <h1>{{ formConfig.name }}</h1>

      <p v-if="formConfig.description">
        {{ formConfig.description }}
      </p>
    </template>

    <div v-show="successMessage">
      <h1>{{ $t("Form submitted") }}</h1>
      <p>{{ successMessage }}</p>
      <ul>
        <li>
          <a href="#" @click.prevent="resetForm">{{ $t("Use again") }}</a>
        </li>
        <li>
          <router-link to="/">{{ $t("Back to home page") }}</router-link>
        </li>
      </ul>
    </div>

    <div v-show="!successMessage">
      <FormErrors :errors="errors" v-if="errors.length > 0" />

      <form v-if="schema" v-on:submit.prevent="submitForm($event)" ref="form">
        <NewForm :schema="schema" />
        <button>{{ $t("Submit") }}</button>
      </form>
    </div>
  </div>
</template>

<script lang="ts">
import axios from "axios";
import { defineComponent, type PropType } from "vue";

import NewForm from "./NewForm.vue";
import type { APIResponseMessage, FormConfig, Schema } from "../interfaces";
import { convertFormValue, parseErrorResponse } from "@/utils";
import FormErrors from "./FormErrors.vue";

const BASE_URL = import.meta.env.VITE_APP_BASE_URI;

export default defineComponent({
  props: {
    formSlug: {
      type: String as PropType<string>,
      required: true,
    },
    schema: {
      type: Object as PropType<Schema>,
      required: true,
    },
  },
  components: {
    NewForm,
    FormErrors,
  },
  data() {
    return {
      errors: [] as string[],
      formConfig: undefined as FormConfig | undefined,
      successMessage: null,
    };
  },
  watch: {
    async formSlug() {
      await this.fetchFormConfig();
    },
  },
  methods: {
    resetForm() {
      const form = this.$refs.form as HTMLFormElement;
      form.reset();
      this.successMessage = null;
      this.errors = [];
    },
    async fetchFormConfig() {
      const response = await this.$store.dispatch(
        "fetchFormConfig",
        this.formSlug
      );
      this.formConfig = response.data;
    },
    async submitForm(event: any) {
      const form = new FormData(event.target);

      const json: { [key: string]: any } = {};
      for (const i of form.entries()) {
        const key = i[0];
        let value: any = i[1];

        json[key] = convertFormValue({
          key,
          value,
          schema: this.schema,
        });
      }

      try {
        var response = await axios.post(
          `${BASE_URL}forms/${this.formSlug}/`,
          json
        );
      } catch (error) {
        var message: APIResponseMessage = {
          contents: "The form has errors.",
          type: "error",
        };
        this.$store.commit("updateApiResponseMessage", message);

        if (axios.isAxiosError(error) && error.response) {
          this.errors = parseErrorResponse(
            error.response.data,
            error.response.status
          );
        }

        return;
      }

      this.errors = [];

      this.successMessage =
        response.data.custom_form_success || "Successfully submitted form";
    },
  },
  async mounted() {
    await this.fetchFormConfig();
  },
});
</script>

<style scoped lang="less">
h1 {
  text-transform: capitalize;
}
</style>
