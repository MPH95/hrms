<template>
	<form class="flex flex-col gap-4 w-full" @submit.prevent="save">
		<label class="flex flex-col gap-1 text-sm text-gray-600">
			{{ __("Log Type") }}
			<select
				v-model="logType"
				class="h-11 rounded border border-gray-200 bg-white px-3 text-base text-gray-900"
				required
				@change="dirty = true"
			>
				<option value="IN">{{ __("IN") }}</option>
				<option value="OUT">{{ __("OUT") }}</option>
			</select>
		</label>
		<label class="flex flex-col gap-1 text-sm text-gray-600">
			{{ __("Time") }}
			<input
				v-model="time"
				type="datetime-local"
				class="h-11 rounded border border-gray-200 bg-white px-3 text-base text-gray-900"
				required
				@change="dirty = true"
			/>
		</label>
		<p v-if="error" class="text-sm text-red-600">{{ error }}</p>
		<Button
			variant="solid"
			class="w-full py-5 text-base"
			:loading="saveCheckin.loading"
			@click="save"
		>
			{{ submitLabel }}
		</Button>
	</form>
</template>

<script setup>
import { computed, inject, ref, watch } from "vue"
import { createResource } from "frappe-ui"

const props = defineProps({
	checkin: {
		type: Object,
		default: null,
	},
	defaultDate: {
		type: String,
		default: "",
	},
	suggestion: {
		type: Object,
		default: null,
	},
})

const emit = defineEmits(["saved"])

const __ = inject("$translate")
const dayjs = inject("$dayjs")

const logType = ref("IN")
const time = ref("")
const error = ref("")
const dirty = ref(false)

const submitLabel = computed(() => {
	if (props.checkin?.name) return __("Save check-in")
	if (logType.value === "OUT") return __("Add check-out")
	return __("Add check-in")
})

const saveCheckin = createResource({
	url: "hr_addon.api.employee_app.save_my_checkin",
})

watch(
	() => props.defaultDate,
	() => {
		dirty.value = false
	}
)

watch(
	() => [props.checkin, props.defaultDate, props.suggestion],
	() => {
		error.value = ""
		if (props.checkin?.time) {
			dirty.value = false
			logType.value = props.checkin.log_type || "IN"
			time.value = dayjs(props.checkin.time).format("YYYY-MM-DDTHH:mm")
			return
		}
		if (!dirty.value && props.suggestion?.time && props.suggestion.action !== "remove") {
			logType.value = props.suggestion.log_type || "IN"
			time.value = dayjs(props.suggestion.time).format("YYYY-MM-DDTHH:mm")
			return
		}
		if (!dirty.value) {
			logType.value = "IN"
			const base = props.defaultDate ? dayjs(props.defaultDate) : dayjs()
			time.value = base.hour(8).minute(0).second(0).format("YYYY-MM-DDTHH:mm")
		}
	},
	{ immediate: true }
)

async function save() {
	error.value = ""
	try {
		await saveCheckin.submit({
			name: props.checkin?.name || null,
			log_type: logType.value,
			time: dayjs(time.value).format("YYYY-MM-DD HH:mm:ss"),
		})
		emit("saved")
	} catch (err) {
		error.value = saveCheckin.error?.messages?.[0] || err?.message || __("Could not save the check-in")
	}
}
</script>
