<template>
  <div class="flex flex-col">
    <TicketBreadcrumbs :parent="route.meta.parent" title="New" />
    <div v-if="template.data?.about" class="mx-5 my-3">
      <div class="prose-f" v-html="sanitize(template.data.about)" />
    </div>
    <div class="grid grid-cols-1 gap-4 px-5 sm:grid-cols-3">
      <UniInput
        v-for="field in visibleFields"
        :key="field.fieldname"
        :field="field"
        :value="templateFields[field.fieldname]"
        @change="templateFields[field.fieldname] = $event.value"
      />
    </div>
    <!-- Ticket Type field -->
    <div class="m-5">
      <label for="ticketType">Ticket Type</label>
      <select
        id="ticketType"
        v-model="ticket_type"
        :disabled="ticket_type_disabled"
      >
        <option value="">Select Ticket Type</option>
        <option value="Mechanical Issue">Mechanical Issue</option>
        <option value="Civil Inquiry">Civil Inquiry</option>
        <option value="Electrical Inquiry">Electrical Inquiry</option>
        <option value="Computer Science">Computer Science</option>
        <option value="General Inquiry">General Inquiry</option>
      </select>
    </div>
    <div class="m-5">
      <FormControl
        v-model="subject"
        type="text"
        label="Subject"
        placeholder="A short description"
      />
    </div>
    <!-- Description field -->
    <div class="m-5">
      <FormControl
        v-model="description"
        type="textarea"
        label="Description"
        placeholder="Detailed explanation"
      />
    </div>

    <!-- Submit Button -->
    <div class="m-5">
      <Button
        label="Submit"
        theme="gray"
        variant="solid"
        :disabled="isSubmitDisabled"
        @click="submitTicket"
      />
    </div>

    <TicketNewArticles :search="subject" class="mx-5 mb-5" />
    <span class="mx-5 mb-5">
      <TicketTextEditor
        ref="editor"
        v-model:attachments="attachments"
        v-model:content="description"
        placeholder="Detailed explanation"
        expand
      />
    </span>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, reactive, watch, defineProps, withDefaults } from "vue";
import { useRoute, useRouter } from "vue-router";
import { createResource, Button, FormControl } from "frappe-ui";
import sanitizeHtml from "sanitize-html";
import { isEmpty } from "lodash";
import { useError } from "@/composables/error";
import { UniInput } from "@/components";

interface P {
  templateId?: string;
}

const props = withDefaults(defineProps<P>(), {
  templateId: "",
});
const route = useRoute();
const router = useRouter();
const subject = ref("");
const description = ref("");
const attachments = ref([]);
const templateFields = reactive({});

const ticket_type = ref("");
const ticket_type_disabled = ref(false); // Indicates whether the ticket type is disabled

const template = createResource({
  url: "helpdesk.helpdesk.doctype.hd_ticket_template.api.get_one",
  makeParams: () => ({
    name: props.templateId || "Default",
  }),
  auto: true,
});

const visibleFields = computed(() =>
  template.data?.fields.filter((f) => route.meta.agent || !f.hide_from_customer)
);
const ticket = createResource({
  url: "helpdesk.helpdesk.doctype.hd_ticket.api.new",
  debounce: 300,
  makeParams: () => ({
    doc: {
      ticket_type: ticket_type.value,
      description: description.value,
      subject: subject.value,
      template: props.templateId,
      ...templateFields,
    },
    attachments: attachments.value,
  }),
  validate: (params) => {
    const fields = visibleFields.value.filter((f) => f.required);
    const toVerify = [
      ...fields.map((f) => f.fieldname),
      "subject",
      "description",
      "ticket_type",
    ];
    for (const field of toVerify) {
      if (isEmpty(params.doc[field])) {
        return `${field} is required`;
      }
    }
    return ""; // Return an empty string if validation passes
  },
  onSuccess: (data) => {
    router.push({
      name: route.meta.onSuccessRoute as string,
      params: {
        ticketId: data.name,
      },
    });
  },
  onError: useError(),
});

function sanitize(html: string) {
  return sanitizeHtml(html, {
    allowedTags: sanitizeHtml.defaults.allowedTags.concat(["img"]),
  });
}
watch(template.data, (newVal) => {
  if (newVal && newVal.ticket_type) {
    ticket_type.value = newVal.ticket_type;
    ticket_type_disabled.value = true;
  }
});

function submitTicket() {
  ticket.submit();
}
</script>
