<template>
	<CustomIonModal :isOpen="open" @did-dismiss="emit('close')">
		<template #actionSheet>
			<div class="bg-white w-full max-w-xl mx-auto flex flex-col gap-5 px-4 pt-5 pb-6 standalone:pb-safe-bottom">
				<div class="flex items-start justify-between gap-3">
					<div>
						<h2 class="text-xl font-semibold text-gray-900">{{ title }}</h2>
						<p v-if="existing" class="text-sm mt-0.5" :class="statusTone">{{ statusLabel }}</p>
					</div>
					<Button variant="ghost" class="!px-1" @click="dismiss">
						<FeatherIcon name="x" class="h-5 w-5" />
					</Button>
				</div>

				<div v-if="loading" class="text-sm text-gray-500 py-6 text-center">{{ __("Loading") }}</div>

				<template v-else>
					<div class="grid gap-2" :class="kinds.length === 3 ? 'grid-cols-3' : 'grid-cols-2'">
						<button
							v-for="option in kinds"
							:key="option.value"
							type="button"
							class="flex flex-col items-center gap-1.5 rounded-xl border px-2 py-3 text-sm font-medium"
							:class="kind === option.value ? option.active : 'border-gray-200 text-gray-700'"
							:disabled="readOnly || Boolean(existing)"
							@click="kind = option.value"
						>
							<FeatherIcon :name="option.icon" class="h-5 w-5" />
							{{ option.label }}
						</button>
					</div>

					<div v-if="kind !== 'home_office' && typeChoices.length > 1" class="flex flex-col gap-1.5">
						<label class="text-sm text-gray-600">{{ __("Leave type") }}</label>
						<select
							v-model="leaveType"
							:disabled="readOnly"
							class="rounded-lg border border-gray-300 bg-white pl-3 pr-9 py-2.5 text-base text-gray-900"
						>
							<option v-for="option in typeChoices" :key="option.leave_type" :value="option.leave_type">
								{{ optionLabel(option) }}
							</option>
						</select>
					</div>
					<p v-else-if="kind !== 'home_office' && typeChoices.length === 1" class="text-sm text-gray-600 -mt-2">
						{{ optionLabel(typeChoices[0]) }}
					</p>
					<p v-if="kind !== 'home_office' && !typeChoices.length" class="text-sm text-orange-700 -mt-2">
						{{ __("No leave type is available for this date. Ask HR to allocate one.") }}
					</p>

					<div class="grid grid-cols-2 gap-3">
						<div class="flex flex-col gap-1.5">
							<label class="text-sm text-gray-600">{{ __("From") }}</label>
							<input
								v-model="fromDate"
								type="date"
								:disabled="readOnly"
								class="rounded-lg border border-gray-300 px-3 py-2.5 text-base text-gray-900"
							/>
						</div>
						<div class="flex flex-col gap-1.5">
							<label class="text-sm text-gray-600">{{ __("To") }}</label>
							<input
								v-model="toDate"
								type="date"
								:min="fromDate"
								:disabled="readOnly"
								class="rounded-lg border border-gray-300 px-3 py-2.5 text-base text-gray-900"
							/>
						</div>
					</div>
					<p class="text-sm text-gray-500 -mt-3">{{ rangeLabel }}</p>

					<div class="flex flex-col gap-2">
						<label class="inline-flex items-center gap-2 text-base text-gray-800">
							<input v-model="halfDay" type="checkbox" :disabled="readOnly" class="h-4 w-4 rounded" />
							{{ __("Half day") }}
						</label>
						<select
							v-if="halfDay && dayCount > 1"
							v-model="halfDayDate"
							:disabled="readOnly"
							class="rounded-lg border border-gray-300 bg-white pl-3 pr-9 py-2 text-sm text-gray-900"
						>
							<option v-for="date in rangeDates" :key="date" :value="date">
								{{ dayjs(date).format("ddd, D MMM") }}
							</option>
						</select>
					</div>

					<div class="flex flex-col gap-1.5">
						<label class="text-sm text-gray-600">{{ __("Note (optional)") }}</label>
						<textarea
							v-model="note"
							rows="2"
							:disabled="readOnly"
							:placeholder="notePlaceholder"
							class="rounded-lg border border-gray-300 px-3 py-2 text-base text-gray-900"
						/>
					</div>

					<ErrorMessage :message="errorMessage" />

					<div v-if="readOnly" class="rounded-xl bg-gray-50 text-gray-700 text-sm px-4 py-3">
						{{ __("This request is already decided. Ask HR if it needs to change.") }}
					</div>
					<div v-else class="flex flex-col gap-2">
						<Button
							variant="solid"
							class="w-full py-5 text-base"
							:loading="saving"
							:disabled="!canSave"
							@click="save"
						>
							{{ existing ? __("Save changes") : __("Send request") }}
						</Button>
						<Button
							v-if="existing"
							variant="subtle"
							theme="red"
							class="w-full py-5 text-base"
							:loading="withdrawing"
							@click="withdraw"
						>
							{{ confirmWithdraw ? __("Tap again to withdraw") : __("Withdraw request") }}
						</Button>
					</div>
				</template>
			</div>
		</template>
	</CustomIonModal>
</template>

<script setup>
import { computed, inject, ref, watch } from "vue"
import { modalController } from "@ionic/vue"
import { call, ErrorMessage, FeatherIcon } from "frappe-ui"

import CustomIonModal from "@/components/CustomIonModal.vue"

const props = defineProps({
	open: { type: Boolean, default: false },
	fromDate: { type: String, required: false },
	toDate: { type: String, required: false },
	initialKind: { type: String, default: "leave" },
	// { kind, name } of an existing request to show or edit
	request: { type: Object, required: false },
})
const emit = defineEmits(["close", "saved", "deleted"])

const __ = inject("$translate")
const dayjs = inject("$dayjs")

const kind = ref("leave")
const leaveType = ref("")
const fromDate = ref("")
const toDate = ref("")
const halfDay = ref(false)
const halfDayDate = ref("")
const note = ref("")
const existing = ref(null)
const leaveTypes = ref([])
const loading = ref(false)
const saving = ref(false)
const withdrawing = ref(false)
const confirmWithdraw = ref(false)
const errorMessage = ref("")

const readOnly = computed(() => Boolean(existing.value) && !existing.value.editable)
const hasIllnessType = computed(() => leaveTypes.value.some((option) => option.illness))

const kinds = computed(() => {
	const list = [
		{ value: "leave", label: __("Vacation"), icon: "sun", active: "border-blue-400 bg-blue-50 text-blue-900" },
		{ value: "sick", label: __("Sick"), icon: "thermometer", active: "border-red-400 bg-red-50 text-red-900" },
		{ value: "home_office", label: __("Home office"), icon: "home", active: "border-purple-400 bg-purple-50 text-purple-900" },
	]
	return hasIllnessType.value || kind.value === "sick" ? list : list.filter((item) => item.value !== "sick")
})

const typeChoices = computed(() => {
	if (kind.value === "home_office") return []
	const wantIllness = kind.value === "sick"
	const matching = leaveTypes.value.filter((option) => option.illness === wantIllness)
	if (existing.value?.leave_type && !matching.some((option) => option.leave_type === existing.value.leave_type)) {
		matching.unshift({ leave_type: existing.value.leave_type, illness: wantIllness, balance: null })
	}
	return matching
})

const title = computed(() => {
	if (!existing.value) return __("New request")
	if (existing.value.kind === "home_office") return __("Home office")
	return existing.value.leave_type
})

const statusLabel = computed(() => {
	const request = existing.value
	if (!request) return ""
	if (request.docstatus === 1) return __("Approved")
	if (request.status === "Rejected") return __("Rejected")
	return __("Waiting for approval")
})

const statusTone = computed(() =>
	existing.value?.docstatus === 1 ? "text-green-700" : "text-orange-700"
)

const rangeDates = computed(() => {
	if (!fromDate.value || !toDate.value) return []
	const start = dayjs(fromDate.value)
	const days = dayjs(toDate.value).diff(start, "day")
	if (days < 0 || days > 366) return []
	return Array.from({ length: days + 1 }, (_, index) => start.add(index, "day").format("YYYY-MM-DD"))
})

const dayCount = computed(() => rangeDates.value.length)

const leaveDays = ref(null)

const rangeLabel = computed(() => {
	if (!dayCount.value) return __("The end date must be on or after the start date.")
	const days = dayCount.value === 1 ? __("1 calendar day") : __("{0} calendar days", [dayCount.value])
	if (kind.value === "home_office" || leaveDays.value === null) return days
	if (Number(leaveDays.value) <= 0) {
		return __("These are not working days in your weekly hours, or they are holidays. No leave is deducted, so there is nothing to request.")
	}
	const booked = Number(leaveDays.value).toFixed(1).replace(/\.0$/, "")
	return `${days} · ${__("counts as {0} leave days", [booked])}`
})

let previewToken = 0
async function previewLeaveDays() {
	const token = ++previewToken
	if (kind.value === "home_office" || !leaveType.value || !dayCount.value) {
		leaveDays.value = null
		return
	}
	try {
		const data = await call("hr_addon.api.my_requests.preview_leave_days", {
			leave_type: leaveType.value,
			from_date: fromDate.value,
			to_date: toDate.value,
			half_day: halfDay.value ? 1 : 0,
			half_day_date: halfDay.value ? halfDayDate.value || fromDate.value : null,
		})
		if (token === previewToken) leaveDays.value = data?.days ?? null
	} catch (error) {
		if (token === previewToken) leaveDays.value = null
	}
}

const notePlaceholder = computed(() => {
	if (kind.value === "sick") return __("You don't need to give a reason")
	if (kind.value === "home_office") return __("e.g. waiting for a delivery")
	return __("e.g. summer holiday")
})

const canSave = computed(() => {
	if (!dayCount.value) return false
	if (kind.value === "home_office") return true
	if (leaveDays.value !== null && Number(leaveDays.value) <= 0) return false
	return Boolean(leaveType.value)
})

function formatDays(value) {
	return Number(value || 0).toFixed(1).replace(/\.0$/, "")
}

function optionLabel(option) {
	const waiting = Number(option.waiting || 0)
	if (option.balance === null || option.balance === undefined) {
		return waiting ? __("{0} · {1} waiting", [option.leave_type, formatDays(waiting)]) : option.leave_type
	}
	const left = formatDays(option.available ?? option.balance)
	if (waiting) return __("{0} · {1} left · {2} waiting", [option.leave_type, left, formatDays(waiting)])
	return __("{0} · {1} left", [option.leave_type, left])
}

function pickLeaveType() {
	if (typeChoices.value.some((option) => option.leave_type === leaveType.value)) return
	leaveType.value = typeChoices.value[0]?.leave_type || ""
}

async function loadOptions(date) {
	const data = await call("hr_addon.api.my_requests.get_request_options", {
		from_date: date,
		exclude: existing.value?.name || null,
	})
	leaveTypes.value = data?.leave_types || []
}

async function reset() {
	errorMessage.value = ""
	confirmWithdraw.value = false
	existing.value = null
	loading.value = true
	try {
		if (props.request?.name) {
			const request = await call("hr_addon.api.my_requests.get_my_request", {
				kind: props.request.kind,
				name: props.request.name,
			})
			existing.value = request
			kind.value = request.kind === "leave" ? (request.illness ? "sick" : "leave") : "home_office"
			fromDate.value = request.from_date
			toDate.value = request.to_date
			halfDay.value = Boolean(request.half_day)
			halfDayDate.value = request.half_day_date || request.from_date
			note.value = request.note || ""
			leaveType.value = request.leave_type || ""
		} else {
			kind.value = props.initialKind
			fromDate.value = props.fromDate || dayjs().format("YYYY-MM-DD")
			toDate.value = props.toDate || fromDate.value
			halfDay.value = false
			halfDayDate.value = fromDate.value
			note.value = ""
			leaveType.value = ""
		}
		await loadOptions(fromDate.value)
		pickLeaveType()
	} catch (error) {
		errorMessage.value = messageOf(error)
	} finally {
		loading.value = false
	}
}

function messageOf(error) {
	const message = error?.messages?.[0] || error?.message || __("Something went wrong")
	return String(message).replace(/<[^>]+>/g, "")
}

async function save() {
	errorMessage.value = ""
	saving.value = true
	try {
		const saved = await call("hr_addon.api.my_requests.save_my_request", {
			kind: kind.value === "home_office" ? "home_office" : "leave",
			from_date: fromDate.value,
			to_date: toDate.value,
			leave_type: kind.value === "home_office" ? null : leaveType.value,
			half_day: halfDay.value ? 1 : 0,
			half_day_date: halfDay.value ? halfDayDate.value || fromDate.value : null,
			note: note.value,
			name: existing.value?.name || null,
		})
		emit("saved", saved)
		dismiss()
	} catch (error) {
		errorMessage.value = messageOf(error)
	} finally {
		saving.value = false
	}
}

async function withdraw() {
	if (!confirmWithdraw.value) {
		confirmWithdraw.value = true
		return
	}
	errorMessage.value = ""
	withdrawing.value = true
	try {
		await call("hr_addon.api.my_requests.delete_my_request", {
			kind: existing.value.kind,
			name: existing.value.name,
		})
		emit("deleted", existing.value)
		dismiss()
	} catch (error) {
		errorMessage.value = messageOf(error)
	} finally {
		withdrawing.value = false
	}
}

function dismiss() {
	modalController.dismiss()
}

watch(
	() => props.open,
	(open) => {
		if (open) reset()
	},
	{ immediate: true }
)

watch(kind, () => {
	if (!loading.value) pickLeaveType()
})

watch([kind, leaveType, fromDate, toDate, halfDay, halfDayDate], () => {
	if (!loading.value) previewLeaveDays()
})

watch(loading, (value) => {
	if (!value) previewLeaveDays()
})

watch(fromDate, (value, previous) => {
	if (!value) return
	if (!toDate.value || toDate.value < value) toDate.value = value
	if (!rangeDates.value.includes(halfDayDate.value)) halfDayDate.value = value
	if (previous && !loading.value && dayjs(value).year() !== dayjs(previous).year()) {
		loadOptions(value).then(pickLeaveType)
	}
})
</script>
