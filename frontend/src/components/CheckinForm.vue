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
			type="button"
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
	if (!time.value) {
		error.value = __("Choose a time.")
		return
	}
	const stamp = dayjs(time.value)
	if (!stamp.isValid()) {
		error.value = __("Choose a valid time.")
		return
	}
	try {
		await saveCheckin.submit({
			name: props.checkin?.name || null,
			log_type: logType.value,
			time: stamp.format("YYYY-MM-DD HH:mm:ss"),
		})
	} catch (err) {
		error.value = messageFrom(err)
		return
	}
	// The request helper reports a server error without rejecting the promise.
	// Desk notes such as "no hour variance" are not a failed save: the punch is
	// already stored, so the day still has to reload.
	const problem = saveCheckin.error ? problemMessage(saveCheckin.error) : ""
	if (problem) {
		error.value = problem
		return
	}
	emit("saved")
}

function problemMessage(err) {
	const raw = err?.messages?.length ? err.messages : [err?.message]
	const problems = raw.filter((message) => message && !isDeskNote(message))
	if (!problems.length && raw.some(Boolean) && raw.every((message) => !message || isDeskNote(message))) {
		return ""
	}
	return problems[0] || __("Could not save the check-in")
}

function isDeskNote(message) {
	const text = String(message)
	return (
		text.includes("hour variance") ||
		text.includes("Overtime Ledger Entry") ||
		/Attendance .+ updated with /.test(text) ||
		/Attendance .+ created/.test(text) ||
		text.includes("SAVEPOINT workday_checkin_sync")
	)
}
</script>
