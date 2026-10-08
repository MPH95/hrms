<template>
	<ion-page>
		<ion-header class="ion-no-border app-shell-header">
			<div class="w-full app-shell">
				<div class="flex flex-row bg-white shadow-sm py-4 px-3 items-center border-b">
					<Button variant="ghost" class="!px-1 mr-1 hover:bg-white" @click="router.back()">
						<FeatherIcon name="chevron-left" class="h-5 w-5" />
					</Button>
					<div>
						<h2 class="text-xl font-semibold text-gray-900">
							{{ dayjs(date).format("ddd, D MMM YYYY") }}
						</h2>
						<div
							v-if="day.data"
							class="text-xs"
							:class="day.data.needs_attention ? 'text-orange-700' : 'text-gray-500'"
						>
							{{ __(day.data.status) }}
							<span class="whitespace-pre"> &middot; </span>
							<span v-if="day.data.completely_missing || isFuture">
								{{ __("Target {0}", [`${Number(day.data.target_hours || 0).toFixed(2)} h`]) }}
							</span>
							<span v-else>{{ Number(day.data.actual_working_hours || 0).toFixed(2) }} h</span>
						</div>
					</div>
				</div>
			</div>
		</ion-header>
		<ion-content class="app-shell-content">
			<div class="flex flex-col min-h-full w-full app-shell p-4 gap-5">
				<div class="flex flex-col gap-5 w-full max-w-2xl mx-auto">
					<div
						v-if="day.data?.schedule === 'free'"
						class="rounded-xl bg-gray-50 text-gray-700 text-sm px-4 py-3"
					>
						{{
							__(
								"This is a free day in your working hours (0 h target). Check-ins here count as overtime in your total."
							)
						}}
					</div>
					<div
						v-else-if="day.data?.schedule === 'none'"
						class="rounded-xl bg-orange-50 text-orange-900 text-sm px-4 py-3"
					>
						{{
							__(
								"This weekday is not in your Weekly Working Hours, so check-ins here are saved but not counted. Ask HR to add this day with 0 hours."
							)
						}}
					</div>
					<div
						v-else-if="day.data?.completely_missing"
						class="rounded-xl bg-orange-50 text-orange-900 text-sm px-4 py-3"
					>
						<div class="font-semibold">{{ __("Nothing was recorded") }}</div>
						<p class="mt-1">
							{{
								__(
									"This was a working day, not a holiday, and there is no check-in. Add the punches if you worked, or send a leave or sick request below."
								)
							}}
						</p>
					</div>
					<div
						v-else-if="suggestion && !isFuture"
						class="rounded-xl bg-orange-50 text-orange-800 text-sm px-3 py-2"
					>
						{{ suggestion.reason }}
					</div>
					<div
						v-else-if="day.data?.needs_attention"
						class="rounded-xl bg-orange-50 text-orange-800 text-sm px-3 py-2"
					>
						{{ __("This day is missing a check-in or check-out. Add the punch you forgot, or correct the time.") }}
					</div>

					<div v-if="requests.length" class="flex flex-col gap-2">
						<div class="text-sm font-medium text-gray-500">{{ __("Requests") }}</div>
						<button
							v-for="request in requests"
							:key="request.name"
							type="button"
							class="flex items-center justify-between gap-3 rounded-xl px-4 py-3 text-left"
							:class="request.pending ? `border border-dashed ${request.border} ${request.soft}` : request.tone"
							@click="openRequest(request)"
						>
							<div class="flex items-center gap-3">
								<FeatherIcon :name="request.icon" class="h-5 w-5 shrink-0" />
								<div>
									<div class="text-sm font-semibold">{{ request.title }}</div>
									<div class="text-xs mt-0.5">{{ request.subtitle }}</div>
								</div>
							</div>
							<div class="flex items-center gap-1 text-xs font-medium shrink-0">
								{{ request.pending ? __("Edit") : __("View") }}
								<FeatherIcon name="chevron-right" class="h-4 w-4" />
							</div>
						</button>
					</div>

					<div v-if="isFuture" class="rounded-xl bg-gray-50 text-gray-600 text-sm px-4 py-3">
						{{
							requestKinds.length
								? __("Check-ins can only be added for today or earlier. You can already plan vacation or home office for this day.")
								: __("Check-ins can only be added for today or earlier.")
						}}
					</div>

					<template v-else>
						<div class="flex flex-col gap-2">
							<div class="text-sm font-medium text-gray-500">{{ __("Check-ins") }}</div>
							<div v-if="!day.data?.checkins?.length" class="text-sm text-gray-500">
								{{ __("No check-ins on this day") }}
							</div>
							<button
								v-for="checkin in day.data?.checkins || []"
								:key="checkin.name"
								type="button"
								class="flex items-center justify-between bg-white rounded px-4 py-3 text-left"
								@click="edit(checkin)"
							>
								<div>
									<div class="text-base text-gray-900">{{ checkin.log_type || __("Check-in") }}</div>
									<div class="text-xs text-gray-500">{{ dayjs(checkin.time).format("HH:mm") }}</div>
								</div>
								<FeatherIcon name="edit-2" class="h-4 w-4 text-gray-500" />
							</button>
						</div>

						<div v-if="suggestion?.action === 'remove'" class="bg-white rounded p-4 flex flex-col gap-3">
							<div class="text-base font-semibold text-gray-900">{{ __("Suggested fix") }}</div>
							<p v-if="removeError" class="text-sm text-red-600">{{ removeError }}</p>
							<Button
								variant="solid"
								class="w-full py-5 text-base"
								:loading="removeCheckin.loading"
								@click="removePunch"
							>
								{{ __("Remove duplicate") }}
							</Button>
							<Button variant="ghost" class="w-full" @click="ignoreSuggestion = true">
								{{ __("Add a punch instead") }}
							</Button>
						</div>
						<div v-else-if="!hasFullDayLeave" class="bg-white rounded-xl p-4">
							<div class="text-base font-semibold text-gray-900">
								{{ day.data?.completely_missing ? __("I worked") : formTitle }}
							</div>
							<p v-if="day.data?.completely_missing" class="text-sm text-gray-500 mt-1 mb-3">
								{{ __("Add the check-in you forgot. A check-out is suggested from your working hours after that.") }}
							</p>
							<div v-else class="mb-3" />
							<CheckinForm
								:key="`${date}-${formCheckin?.name || suggestion?.action || 'add'}-${formCheckin?.time || suggestion?.time || ''}`"
								:checkin="formCheckin"
								:suggestion="formCheckin ? null : suggestion"
								:defaultDate="date"
								@saved="onSaved"
							/>
							<Button v-if="formCheckin" variant="ghost" class="w-full mt-2" @click="addAnother">
								{{ __("Add another instead") }}
							</Button>
						</div>
					</template>

					<div v-if="requestKinds.length" class="bg-white rounded-xl p-4 flex flex-col gap-3">
						<div>
							<div class="text-base font-semibold text-gray-900">
								{{ day.data?.completely_missing ? __("I was away") : __("Plan this day") }}
							</div>
							<p class="text-sm text-gray-500 mt-1">
								{{
									__(
										"Requests go to your approver. Until then they show as waiting in your calendar and you can still change them."
									)
								}}
							</p>
						</div>
						<div class="grid gap-2" :class="requestKinds.length === 3 ? 'grid-cols-3' : 'grid-cols-2'">
							<button
								v-for="option in requestKinds"
								:key="option.kind"
								type="button"
								class="flex flex-col items-center gap-1.5 rounded-xl border px-2 py-3 text-sm font-medium"
								:class="option.tone"
								@click="newRequest(option.kind)"
							>
								<FeatherIcon :name="option.icon" class="h-5 w-5" />
								{{ option.label }}
							</button>
						</div>
					</div>
				</div>
			</div>
		</ion-content>

		<RequestSheet
			:open="sheet.open"
			:fromDate="date"
			:toDate="date"
			:initialKind="sheet.kind"
			:request="sheet.request"
			@close="sheet.open = false"
			@saved="(request) => onRequestChanged(request, __('Request sent'))"
			@deleted="(request) => onRequestChanged(request, __('Request withdrawn'))"
		/>
	</ion-page>
</template>

<script setup>
import { computed, inject, reactive, ref, watch } from "vue"
import { useRouter } from "vue-router"
import { IonPage, IonHeader, IonContent, onIonViewWillEnter } from "@ionic/vue"
import { createResource, FeatherIcon, toast } from "frappe-ui"

import CheckinForm from "@/components/CheckinForm.vue"
import RequestSheet from "@/components/RequestSheet.vue"

const props = defineProps({
	date: {
		type: String,
		required: true,
	},
})

const __ = inject("$translate")
const dayjs = inject("$dayjs")
const router = useRouter()
const editing = ref(null)
const ignoreSuggestion = ref(false)
const removeError = ref("")
const sheet = reactive({ open: false, kind: "leave", request: null })

const day = createResource({
	url: "hr_addon.api.employee_app.get_my_day",
})

const removeCheckin = createResource({
	url: "hr_addon.api.employee_app.delete_my_checkin",
})

const isFuture = computed(() => dayjs(props.date).isAfter(dayjs(), "day"))

const suggestion = computed(() => (ignoreSuggestion.value ? null : day.data?.suggestion || null))

const hasFullDayLeave = computed(() => {
	const data = day.data
	return (
		Boolean(data?.leave) &&
		["On Leave", "Pending Leave"].includes(data.status) &&
		!data.checkins?.length
	)
})

const requests = computed(() => {
	const data = day.data
	if (!data) return []
	const list = []
	if (data.leave) {
		const pending = data.leave.docstatus === 0
		const illness = data.leave.illness
		list.push({
			kind: "leave",
			name: data.leave.name,
			pending,
			icon: illness ? "thermometer" : "sun",
			title: illness
				? __("Sick · {0}", [data.leave.leave_type])
				: __("Vacation · {0}", [data.leave.leave_type]),
			subtitle: data.schedule === "free"
				? __("Not deducted. This is not a working day in your weekly hours, so it does not count as leave.")
				: pending
					? __("Waiting for approval · tap to change or withdraw")
					: __("Approved"),
			tone: illness ? "bg-red-50 text-red-900" : "bg-blue-50 text-blue-900",
			soft: illness ? "bg-red-50/40 text-red-900" : "bg-blue-50/40 text-blue-900",
			border: illness ? "border-red-300" : "border-blue-300",
		})
	}
	if (data.home_office) {
		const pending = data.home_office.docstatus === 0
		list.push({
			kind: "home_office",
			name: data.home_office.name,
			pending,
			icon: "home",
			title: data.home_office.half_day ? __("Home office · half day") : __("Home office"),
			subtitle: pending
				? __("Waiting for approval · check-ins count as usual")
				: __("Approved · check-ins count as usual"),
			tone: "bg-purple-50 text-purple-900",
			soft: "bg-purple-50/40 text-purple-900",
			border: "border-purple-300",
		})
	}
	return list
})

const requestKinds = computed(() => {
	const data = day.data
	if (!data || data.status === "Holiday" || hasFullDayLeave.value) return []
	const expected = data.schedule === "work" || Boolean(data.checkins?.length)
	if (!expected) return []
	const kinds = []
	if (!data.leave) {
		kinds.push({ kind: "leave", label: __("Vacation"), icon: "sun", tone: "border-blue-200 text-blue-900" })
		if (!isFuture.value || dayjs(props.date).isSame(dayjs().add(1, "day"), "day")) {
			kinds.push({ kind: "sick", label: __("Sick"), icon: "thermometer", tone: "border-red-200 text-red-900" })
		}
	}
	if (!data.home_office && !hasFullDayLeave.value) {
		kinds.push({ kind: "home_office", label: __("Home office"), icon: "home", tone: "border-purple-200 text-purple-900" })
	}
	return kinds
})

const formCheckin = computed(() => {
	if (editing.value) return editing.value
	if (suggestion.value?.action === "correct") {
		return {
			name: suggestion.value.name,
			log_type: suggestion.value.log_type,
			time: suggestion.value.time,
		}
	}
	return null
})

const formTitle = computed(() => {
	if (formCheckin.value?.name) return __("Correct check-in")
	if (day.data?.schedule === "free" && !day.data.checkins?.length) return __("Add overtime")
	if (suggestion.value?.log_type === "OUT") return __("Suggested check-out")
	if (suggestion.value?.log_type === "IN") return __("Suggested check-in")
	return day.data?.checkins?.length ? __("Add check-in") : __("Add missing check-in")
})

watch(
	() => props.date,
	(date) => {
		editing.value = null
		ignoreSuggestion.value = false
		removeError.value = ""
		sheet.open = false
		day.submit({ date })
	},
	{ immediate: true }
)

function edit(checkin) {
	ignoreSuggestion.value = true
	editing.value = { ...checkin }
}

function addAnother() {
	ignoreSuggestion.value = true
	editing.value = null
}

onIonViewWillEnter(() => {
	day.submit({ date: props.date })
})

async function onSaved() {
	editing.value = null
	ignoreSuggestion.value = false
	removeError.value = ""
	await day.submit({ date: props.date })
}

function newRequest(kind) {
	sheet.request = null
	sheet.kind = kind
	sheet.open = true
}

function openRequest(request) {
	sheet.request = { kind: request.kind, name: request.name }
	sheet.open = true
}

function onRequestChanged(request, message) {
	sheet.open = false
	toast({
		title: message,
		icon: "check-circle",
		position: "bottom-center",
		iconClasses: "text-green-500",
	})
	returnToCalendar(request?.from_date || props.date)
}

function returnToCalendar(date) {
	const back = String(router.options.history.state?.back || "")
	const cameFromCalendar = (back.startsWith("/workdays") && !/\/workdays\/\d/.test(back)) || back.startsWith("/team")
	if (cameFromCalendar) {
		router.back()
		return
	}
	router.replace({ name: "WorkdayListView", query: { month: dayjs(date).format("YYYY-MM") } })
}

async function removePunch() {
	removeError.value = ""
	try {
		await removeCheckin.submit({ name: suggestion.value?.name })
	} catch (err) {
		const message = problemMessage(removeCheckin.error || err)
		if (message) {
			removeError.value = message
			return
		}
	}
	if (removeCheckin.error) {
		const message = problemMessage(removeCheckin.error)
		if (message) {
			removeError.value = message
			return
		}
	}
	onSaved()
}

function problemMessage(err) {
	const raw = err?.messages?.length ? err.messages : [err?.message]
	const problems = raw.filter((message) => message && !isDeskNote(message))
	if (!problems.length && raw.some((message) => message && isDeskNote(message))) return ""
	return problems[0] || __("Could not remove the check-in")
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
