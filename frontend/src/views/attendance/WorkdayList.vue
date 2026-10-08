<template>
	<ion-page>
		<ion-header class="ion-no-border app-shell-header">
			<div class="w-full app-shell">
				<div class="flex flex-row bg-white shadow-sm py-4 px-3 items-center justify-between border-b">
					<div class="flex flex-row items-center">
						<Button variant="ghost" class="!px-1 mr-1 hover:bg-white" @click="router.back()">
							<FeatherIcon name="chevron-left" class="h-5 w-5" />
						</Button>
						<h2 class="text-xl font-semibold text-gray-900">{{ __("Workdays") }}</h2>
					</div>
					<div class="flex flex-row items-center gap-2">
						<Button variant="ghost" class="!px-2" @click="shiftMonth(-1)">
							<FeatherIcon name="chevron-left" class="h-5 w-5" />
						</Button>
						<button
							type="button"
							class="text-sm font-medium text-gray-800 min-w-28 text-center"
							:title="__('Back to this month')"
							@click="cursor = dayjs().startOf('month')"
						>
							{{ monthLabel }}
						</button>
						<Button variant="ghost" class="!px-2" :disabled="isLastMonth" @click="shiftMonth(1)">
							<FeatherIcon name="chevron-right" class="h-5 w-5" />
						</Button>
					</div>
				</div>
			</div>
		</ion-header>
		<ion-content class="app-shell-content">
			<div class="flex flex-col min-h-full w-full app-shell p-4 gap-4">
				<div class="flex flex-wrap items-center justify-between gap-3">
					<div
						v-if="workdays.data?.missing_days"
						class="rounded-xl bg-orange-50 text-orange-800 text-sm px-3 py-2"
					>
						{{
							workdays.data.missing_days === 1
								? __("1 working day has no check-in or leave")
								: __("{0} working days have no check-in or leave", [workdays.data.missing_days])
						}}
					</div>
					<div v-else />
					<Button variant="solid" @click="planRequest">
						<template #prefix>
							<FeatherIcon name="plus" class="h-4 w-4" />
						</template>
						{{ __("Plan leave or home office") }}
					</Button>
				</div>

				<div class="flex flex-col gap-2 sm:hidden select-none">
					<button
						v-for="day in monthDays"
						:key="day.date"
						type="button"
						class="flex items-center justify-between gap-3 rounded-xl px-4 py-3 text-left border"
						:class="[cellClass(day), inRange(day.date) ? 'ring-2 ring-gray-900 ring-offset-2' : '']"
						@pointerdown="startDrag(day, $event)"
						@pointerenter="dragRange(day)"
						@click="tapDay(day)"
						@dblclick.stop="openDay(day)"
					>
						<div class="flex items-center gap-3">
							<FeatherIcon v-if="cellIcon(day)" :name="cellIcon(day)" class="h-4 w-4 shrink-0" />
							<div>
								<div class="text-base font-medium flex items-center gap-2">
									{{ dayjs(day.date).format("ddd, D MMM") }}
									<span
										v-if="isToday(day)"
										class="rounded-full bg-blue-600 px-2 py-0.5 text-[10px] font-semibold uppercase tracking-wide text-white"
									>
										{{ __("Today") }}
									</span>
								</div>
								<div class="text-xs mt-0.5 opacity-80">{{ cellLabel(day) }}</div>
							</div>
						</div>
						<div v-if="day.actual_working_hours" class="text-sm font-semibold">
							{{ Number(day.actual_working_hours).toFixed(2) }} h
						</div>
					</button>
				</div>

				<div class="hidden sm:block bg-white rounded-2xl p-3 sm:p-5 select-none">
					<div class="grid grid-cols-7 gap-1 sm:gap-2 mb-2">
						<div
							v-for="label in weekdayLabels"
							:key="label"
							class="text-center text-[11px] sm:text-xs font-medium text-gray-500 py-1"
						>
							{{ label }}
						</div>
					</div>
					<div v-if="workdays.loading && !workdays.data" class="text-sm text-gray-500 py-8 text-center">
						{{ __("Loading") }}
					</div>
					<div v-else class="grid grid-cols-7 gap-1 sm:gap-2">
						<div v-for="(cell, index) in cells" :key="cell?.date || `blank-${index}`">
							<button
								v-if="cell"
								type="button"
								class="w-full min-h-14 sm:min-h-20 rounded-xl px-1 py-1.5 flex flex-col items-center justify-center text-center gap-1"
								:class="[cellClass(cell), inRange(cell.date) ? 'ring-2 ring-gray-900 ring-offset-2 z-10' : '']"
								@pointerdown="startDrag(cell, $event)"
								@pointerenter="dragRange(cell)"
								@click="tapDay(cell)"
								@dblclick.stop="openDay(cell)"
							>
								<span
									class="text-sm sm:text-base font-semibold leading-none"
									:class="
										isToday(cell)
											? 'inline-flex h-6 w-6 sm:h-7 sm:w-7 items-center justify-center rounded-full bg-blue-600 text-white'
											: ''
									"
								>
									{{ dayjs(cell.date).date() }}
								</span>
								<FeatherIcon v-if="cellIcon(cell)" :name="cellIcon(cell)" class="h-3.5 w-3.5" />
								<span class="text-[10px] sm:text-xs leading-tight">
									{{ cellLabel(cell) }}
								</span>
							</button>
						</div>
					</div>
				</div>

				<div class="flex flex-wrap gap-x-4 gap-y-2 px-1 text-xs text-gray-600">
					<span v-for="item in legend" :key="item.label" class="inline-flex items-center gap-1.5">
						<span class="h-3.5 w-3.5 rounded" :class="item.tone" />
						{{ item.label }}
					</span>
				</div>
					<p class="px-1 text-xs text-gray-500">
					{{
						__(
							"Tap a day, then the last day (or drag with the mouse) to plan Vacation, Sick or Home office. Double-click a day to open it and fix check-ins."
						)
					}}
				</p>

				<div v-if="attentionDays.length" class="flex flex-col gap-2">
					<div class="text-sm font-medium text-gray-500">{{ __("To resolve") }}</div>
					<button
						v-for="day in attentionDays"
						:key="day.date"
						type="button"
						class="flex items-center justify-between gap-3 bg-white rounded-xl px-4 py-3 text-left border border-orange-200"
						@click="openDay(day)"
					>
						<div>
							<div class="text-base font-medium text-gray-900">
								{{ dayjs(day.date).format("ddd, D MMM") }}
							</div>
							<div class="text-xs text-orange-700 mt-0.5">{{ __(day.status) }}</div>
						</div>
						<FeatherIcon name="chevron-right" class="h-4 w-4 text-gray-400" />
					</button>
				</div>

				<div v-if="range" class="sticky bottom-4 z-30">
					<div class="mx-auto max-w-2xl bg-gray-900 text-white rounded-2xl shadow-xl p-4 flex flex-col gap-3">
						<div class="flex items-start justify-between gap-3">
							<div>
								<div class="text-base font-semibold">{{ rangeTitle }}</div>
								<div class="text-xs text-gray-300 mt-0.5">{{ rangeHint }}</div>
							</div>
							<button type="button" class="p-1 text-gray-300 hover:text-white" @click="clearRange">
								<FeatherIcon name="x" class="h-5 w-5" />
							</button>
						</div>
						<div class="grid grid-cols-3 gap-2">
							<button
								v-for="option in planOptions"
								:key="option.kind"
								type="button"
								class="flex flex-col items-center gap-1 rounded-xl bg-white/10 hover:bg-white/20 px-2 py-2.5 text-sm font-medium"
								@click="plan(option.kind)"
							>
								<FeatherIcon :name="option.icon" class="h-4 w-4" />
								{{ option.label }}
							</button>
						</div>
						<button
							v-if="rangeIsSingleDay"
							type="button"
							class="text-xs text-gray-300 underline self-start"
							@click="openDay({ date: range.start })"
						>
							{{ __("Open this day") }}
						</button>
					</div>
				</div>
			</div>
		</ion-content>

		<RequestSheet
			:open="sheetOpen"
			:fromDate="range?.start || sheetDate"
			:toDate="range?.end || sheetDate"
			:initialKind="sheetKind"
			@close="sheetOpen = false"
			@saved="onRequestSaved"
		/>
	</ion-page>
</template>

<script setup>
import { computed, inject, onBeforeUnmount, onMounted, ref, watch } from "vue"
import { useRoute, useRouter } from "vue-router"
import { IonPage, IonHeader, IonContent, onIonViewWillEnter } from "@ionic/vue"
import { createResource, FeatherIcon, toast } from "frappe-ui"

import RequestSheet from "@/components/RequestSheet.vue"
import { PENDING_BORDER, REQUEST_STYLES, requestClass, requestOnDay } from "@/utils/dayKinds"

const __ = inject("$translate")
const dayjs = inject("$dayjs")
const router = useRouter()
const route = useRoute()

const MONTHS_AHEAD = 12

const cursor = ref(initialMonth())
const sheetOpen = ref(false)
const sheetDate = ref("")
const sheetKind = ref("leave")
const range = ref(null)
const dragging = ref(false)

const planOptions = computed(() => [
	{ kind: "leave", label: __("Vacation"), icon: "sun" },
	{ kind: "sick", label: __("Sick"), icon: "thermometer" },
	{ kind: "home_office", label: __("Home office"), icon: "home" },
])

const monthLabel = computed(() => cursor.value.format("MMMM YYYY"))
const isLastMonth = computed(() =>
	cursor.value.isSame(dayjs().add(MONTHS_AHEAD, "month"), "month")
)
const weekdayLabels = [1, 2, 3, 4, 5, 6, 0].map((day) => dayjs().day(day).format("dd"))

const legend = computed(() => [
	{ label: __("Today"), tone: "bg-blue-600" },
	{ label: __("Worked"), tone: "bg-green-100" },
	{ label: __("Home office"), tone: REQUEST_STYLES.home.solid },
	{ label: __("Vacation"), tone: REQUEST_STYLES.leave.solid },
	{ label: __("Sick"), tone: REQUEST_STYLES.sick.solid },
	{ label: __("Waiting for approval"), tone: `bg-white ${PENDING_BORDER} border-gray-400` },
	{ label: __("Missing"), tone: "bg-orange-100" },
	{ label: __("Holiday"), tone: "bg-gray-200" },
])

const workdays = createResource({
	url: "hr_addon.api.employee_app.get_my_workdays",
})

const cells = computed(() => {
	const start = cursor.value.startOf("month")
	const pad = (start.day() + 6) % 7
	const byDate = Object.fromEntries((workdays.data?.days || []).map((day) => [day.date, day]))
	const blanks = Array.from({ length: pad }, () => null)
	const days = Array.from({ length: start.daysInMonth() }, (_, index) => {
		const date = start.date(index + 1).format("YYYY-MM-DD")
		return byDate[date] || { date, status: "Off", needs_attention: false }
	})
	return [...blanks, ...days]
})

const monthDays = computed(() => cells.value.filter(Boolean))

const attentionDays = computed(() =>
	(workdays.data?.days || []).filter((day) => day.needs_attention)
)

function initialMonth() {
	const month = route.query.month
	const parsed = month ? dayjs(`${month}-01`) : null
	return parsed?.isValid() ? parsed.startOf("month") : dayjs().startOf("month")
}

function load() {
	workdays.submit({
		year: cursor.value.year(),
		month: cursor.value.month() + 1,
	})
}

function shiftMonth(offset) {
	const next = cursor.value.add(offset, "month")
	if (next.isAfter(dayjs().add(MONTHS_AHEAD, "month"), "month")) return
	cursor.value = next
}

function openDay(day) {
	const now = Date.now()
	if (lastOpenedDate === day.date && now - lastOpenedAt < 500) return
	lastOpenedDate = day.date
	lastOpenedAt = now
	router.push({ name: "WorkdayDayView", params: { date: day.date } })
}

let lastOpenedDate = ""
let lastOpenedAt = 0

const rangeIsSingleDay = computed(() => range.value && range.value.start === range.value.end)

const rangeDays = computed(() => {
	if (!range.value) return []
	return monthDays.value.filter((day) => day.date >= range.value.start && day.date <= range.value.end)
})

const rangeTitle = computed(() => {
	if (!range.value) return ""
	const start = dayjs(range.value.start)
	const end = dayjs(range.value.end)
	if (rangeIsSingleDay.value) return start.format("dddd, D MMM")
	return `${start.format("D MMM")} – ${end.format("D MMM")}`
})

const rangeHint = computed(() => {
	const total = rangeDays.value.length
	if (total <= 1) return __("Double-click to open this day, or tap another day to plan a period")
	return __("{0} days selected · tap a day to start over", [total])
})

let dragAnchor = null
let dragMoved = false

function setRange(a, b) {
	range.value = a <= b ? { start: a, end: b } : { start: b, end: a }
}

function inRange(date) {
	return Boolean(range.value) && date >= range.value.start && date <= range.value.end
}

function tapDay(day) {
	if (dragMoved) {
		dragMoved = false
		return
	}
	const current = range.value
	if (current && current.start === current.end) {
		if (day.date === current.start) {
			openDay(day)
			return
		}
		setRange(current.start, day.date)
		return
	}
	setRange(day.date, day.date)
}

function startDrag(day, event) {
	if (event.pointerType !== "mouse") return
	dragAnchor = day.date
	dragMoved = false
	dragging.value = true
}

function dragRange(day) {
	if (!dragging.value || day.date === dragAnchor) return
	dragMoved = true
	setRange(dragAnchor, day.date)
}

function stopDrag() {
	dragging.value = false
	setTimeout(() => (dragMoved = false), 0)
}

function clearRange() {
	range.value = null
	dragging.value = false
}

function plan(kind) {
	sheetKind.value = kind
	sheetOpen.value = true
}

function planRequest() {
	const today = dayjs()
	sheetDate.value = cursor.value.isAfter(today, "month")
		? cursor.value.format("YYYY-MM-DD")
		: today.format("YYYY-MM-DD")
	sheetKind.value = "leave"
	clearRange()
	sheetOpen.value = true
}

function onRequestSaved(request) {
	sheetOpen.value = false
	toast({
		title: __("Request sent"),
		icon: "check-circle",
		position: "bottom-center",
		iconClasses: "text-green-500",
	})
	clearRange()
	const month = dayjs(request?.from_date).startOf("month")
	if (month.isValid() && !month.isSame(cursor.value, "month")) cursor.value = month
	else load()
}

function isToday(day) {
	return dayjs(day.date).isSame(dayjs(), "day")
}

function cellClass(day) {
	const request = requestOnDay(day)
	if (request) return requestClass(request.kind, request.pending)
	const tone = {
		Present: "bg-green-100 text-green-900",
		"Half Day": "bg-green-50 text-green-800",
		Missing: "bg-orange-100 text-orange-900",
		"Missing Checkin": "bg-orange-100 text-orange-900",
		Absent: "bg-orange-100 text-orange-900",
		Holiday: "bg-gray-200 text-gray-600",
		Open: "bg-white text-gray-800 border border-gray-200",
		Off: "bg-gray-50 text-gray-600 border border-dashed border-gray-300",
		"Not Workday": "bg-gray-50 text-gray-600 border border-dashed border-gray-300",
	}[day.status] || "bg-white text-gray-800 border border-gray-200"
	return tone
}

function cellIcon(day) {
	const request = requestOnDay(day)
	return request ? REQUEST_STYLES[request.kind].icon : null
}

function cellLabel(day) {
	const request = requestOnDay(day)
	if (request) {
		const name = { leave: __("Vacation"), sick: __("Sick"), home: __("Home office") }[request.kind]
		const hours = Number(day.actual_working_hours || 0)
		const detail = request.pending ? __("waiting") : hours ? `${hours.toFixed(1)} h` : request.halfDay ? "½" : ""
		return detail ? `${name} · ${detail}` : name
	}
	if (day.status === "Present" || day.status === "Half Day") {
		return `${Number(day.actual_working_hours || 0).toFixed(1)} h`
	}
	if (day.status === "Missing") return __("Missing")
	if (day.status === "Missing Checkin") return __("Open")
	if (day.status === "Holiday") return __("Holiday")
	if (day.status === "Open" && dayjs(day.date).isSame(dayjs(), "day")) return __("Today")
	if (day.status === "Absent") return __("Absent")
	if (day.status === "Off" || day.status === "Not Workday") {
		return day.schedule === "free" ? __("Free") : ""
	}
	return ""
}

watch(cursor, () => {
	clearRange()
	load()
}, { immediate: true })

onMounted(() => window.addEventListener("pointerup", stopDrag))
onBeforeUnmount(() => window.removeEventListener("pointerup", stopDrag))
onIonViewWillEnter(() => {
	if (workdays.data) load()
})
</script>
