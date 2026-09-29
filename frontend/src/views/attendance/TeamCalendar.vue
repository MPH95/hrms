<template>
	<ion-page>
		<ion-header class="ion-no-border app-shell-header">
			<div class="w-full app-shell">
				<div class="flex flex-row bg-white shadow-sm py-4 px-3 items-center justify-between border-b">
					<div class="flex flex-row items-center">
						<Button variant="ghost" class="!px-1 mr-1 hover:bg-white" @click="router.back()">
							<FeatherIcon name="chevron-left" class="h-5 w-5" />
						</Button>
						<h2 class="text-xl font-semibold text-gray-900">{{ __("Team") }}</h2>
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
						<Button variant="ghost" class="!px-2" @click="shiftMonth(1)">
							<FeatherIcon name="chevron-right" class="h-5 w-5" />
						</Button>
					</div>
				</div>
			</div>
		</ion-header>
		<ion-content class="app-shell-content">
			<div class="flex flex-col min-h-full w-full app-shell p-4 lg:px-8 gap-4">
				<div class="flex flex-wrap items-center gap-3">
					<select
						v-model="department"
						class="rounded-lg border border-gray-300 bg-white pl-3 pr-9 py-2 text-sm text-gray-800"
					>
						<option value="">{{ __("All departments") }}</option>
						<option v-for="name in team.data?.departments || []" :key="name" :value="name">
							{{ name }}
						</option>
					</select>
					<div class="text-sm text-gray-600">{{ todaySummary }}</div>
				</div>

				<div class="rounded-xl bg-gray-50 text-gray-700 text-sm px-4 py-3 flex items-start gap-2">
					<FeatherIcon name="info" class="h-4 w-4 mt-0.5 shrink-0" />
					<span>
						{{
							__(
								"Plan your own time in your row: tap a day, then tap the last day of the period (or drag with the mouse). Then choose vacation, sick or home office."
							)
						}}
					</span>
				</div>

				<div class="bg-white rounded-2xl overflow-hidden border border-gray-100">
					<div v-if="team.loading && !team.data" class="text-sm text-gray-500 py-10 text-center">
						{{ __("Loading") }}
					</div>
					<div v-else-if="!members.length" class="text-sm text-gray-500 py-10 text-center">
						{{ __("No employees found") }}
					</div>
					<div v-else ref="scroller" class="overflow-x-auto select-none">
						<table class="border-separate border-spacing-0 text-xs w-full">
							<thead>
								<tr>
									<th
										class="sticky left-0 z-20 bg-white min-w-40 max-w-40 px-3 py-2 text-left font-medium text-gray-500 border-b"
									>
										{{ __("Member") }}
									</th>
									<th
										v-for="date in dates"
										:key="date.key"
										:data-today="date.isToday || undefined"
										class="px-0.5 py-1.5 font-medium border-b text-center"
										:class="[
											date.isToday ? 'text-gray-900' : 'text-gray-500',
											date.isWeekend ? 'bg-gray-50' : 'bg-white',
										]"
									>
										<div class="text-[10px] leading-none">{{ date.weekday }}</div>
										<div
											class="mt-1 mx-auto h-6 w-6 leading-6 rounded-full text-xs"
											:class="date.isToday ? 'bg-gray-900 text-white' : ''"
										>
											{{ date.day }}
										</div>
									</th>
								</tr>
							</thead>
							<tbody>
								<tr v-for="member in members" :key="member.employee" :class="member.is_me ? 'bg-amber-50/40' : ''">
									<td
										class="sticky left-0 z-10 min-w-40 max-w-40 px-3 py-1.5 border-b"
										:class="member.is_me ? 'bg-amber-50' : 'bg-white'"
									>
										<div class="truncate text-sm font-medium text-gray-900">
											{{ member.employee_name }}
										</div>
										<div class="truncate text-[11px]" :class="member.is_me ? 'text-amber-700 font-medium' : 'text-gray-500'">
											{{ member.is_me ? __("You · tap days to plan") : member.department || member.designation || "" }}
										</div>
									</td>
									<td
										v-for="(cell, index) in member.days"
										:key="cell.date"
										class="px-0.5 py-1.5 border-b text-center"
										:class="dates[index]?.isWeekend && !member.is_me ? 'bg-gray-50' : ''"
									>
										<button
											type="button"
											class="h-8 w-8 min-w-8 mx-auto rounded-md flex items-center justify-center"
											:class="[
												member.is_me && inRange(cell.date) ? 'bg-gray-900 text-white' : cellClass(cell),
												!member.is_me && isSelected(member, cell) ? 'ring-2 ring-gray-900' : '',
												member.is_me ? 'cursor-pointer hover:ring-2 hover:ring-gray-300' : '',
											]"
											:title="`${member.employee_name} · ${dayjs(cell.date).format('ddd, D MMM')} · ${describe(cell)}`"
											@pointerdown="member.is_me ? startDrag(cell, $event) : null"
											@pointerenter="member.is_me ? dragRange(cell) : null"
											@click="member.is_me ? tapMyDay(cell) : select(member, cell)"
										>
											<FeatherIcon v-if="icon(cell)" :name="icon(cell)" class="h-3.5 w-3.5" />
											<span v-else-if="cell.half_day" class="text-[10px] font-semibold">½</span>
										</button>
									</td>
								</tr>
							</tbody>
						</table>
					</div>
				</div>

				<div v-if="selected" class="bg-white rounded-xl px-4 py-3 flex items-center justify-between gap-3">
					<div>
						<div class="text-sm font-semibold text-gray-900">
							{{ selected.member.employee_name }} · {{ dayjs(selected.cell.date).format("ddd, D MMM") }}
						</div>
						<div class="text-sm text-gray-600 mt-0.5">{{ describe(selected.cell) }}</div>
					</div>
					<Button variant="ghost" class="!px-1" @click="selected = null">
						<FeatherIcon name="x" class="h-4 w-4" />
					</Button>
				</div>

				<div class="flex flex-wrap gap-x-4 gap-y-2 px-1 text-xs text-gray-600">
					<span v-for="item in legend" :key="item.kind" class="inline-flex items-center gap-1.5">
						<span class="h-4 w-4 rounded flex items-center justify-center" :class="item.tone">
							<FeatherIcon v-if="item.icon" :name="item.icon" class="h-2.5 w-2.5" />
						</span>
						{{ item.label }}
					</span>
					<span class="inline-flex items-center gap-1.5">
						<span class="h-4 w-4 rounded border-2 border-dashed border-gray-400" />
						{{ __("Waiting for approval") }}
					</span>
				</div>

				<div v-if="range" class="sticky bottom-4 z-30 mt-auto">
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
						<div v-if="rangeRequest" class="flex gap-2">
							<button
								type="button"
								class="flex-1 rounded-xl bg-white text-gray-900 px-3 py-2.5 text-sm font-medium"
								@click="openExisting"
							>
								{{ rangeRequest.pending ? __("Change or withdraw request") : __("View request") }}
							</button>
							<button
								v-if="rangeIsSingleDay && !isFutureRange"
								type="button"
								class="rounded-xl bg-white/10 px-3 py-2.5 text-sm font-medium"
								@click="openMyDay(range.start)"
							>
								{{ __("Open day") }}
							</button>
						</div>
						<div v-else class="grid grid-cols-3 gap-2">
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
							v-if="!rangeRequest && rangeIsSingleDay && !isFutureRange"
							type="button"
							class="text-xs text-gray-300 underline self-start"
							@click="openMyDay(range.start)"
						>
							{{ __("Open this day to fix check-ins") }}
						</button>
					</div>
				</div>
			</div>
		</ion-content>

		<RequestSheet
			:open="sheet.open"
			:fromDate="range?.start"
			:toDate="range?.end"
			:initialKind="sheet.kind"
			:request="sheet.request"
			@close="sheet.open = false"
			@saved="(request) => onRequestChanged(request, __('Request sent'))"
			@deleted="(request) => onRequestChanged(request, __('Request withdrawn'))"
		/>
	</ion-page>
</template>

<script setup>
import { computed, inject, nextTick, onBeforeUnmount, onMounted, reactive, ref, watch } from "vue"
import { useRouter } from "vue-router"
import { IonPage, IonHeader, IonContent, onIonViewWillEnter } from "@ionic/vue"
import { createResource, FeatherIcon, toast } from "frappe-ui"

import RequestSheet from "@/components/RequestSheet.vue"

const __ = inject("$translate")
const dayjs = inject("$dayjs")
const router = useRouter()

const cursor = ref(dayjs().startOf("month"))
const department = ref("")
const selected = ref(null)
const scroller = ref(null)
const range = ref(null)
const dragging = ref(false)
const sheet = reactive({ open: false, kind: "leave", request: null })

const KINDS = {
	work: { tone: "bg-green-200 text-green-800", icon: null, label: __("At work") },
	home: { tone: "bg-purple-200 text-purple-800", icon: "home", label: __("Home office") },
	leave: { tone: "bg-blue-100 text-blue-800", icon: "sun", label: __("Vacation") },
	sick: { tone: "bg-red-100 text-red-800", icon: "thermometer", label: __("Sick") },
	holiday: { tone: "bg-gray-200 text-gray-600", icon: "gift", label: __("Holiday") },
	missing: { tone: "bg-orange-100 text-orange-800", icon: "alert-circle", label: __("Not recorded") },
	open: { tone: "bg-white border border-gray-200", icon: null, label: __("Planned workday") },
	free: { tone: "bg-transparent", icon: null, label: __("Free day") },
}

const PENDING_BORDERS = {
	home: "border-purple-400",
	leave: "border-blue-400",
	sick: "border-red-400",
}

const planOptions = computed(() => [
	{ kind: "leave", label: __("Vacation"), icon: "sun" },
	{ kind: "sick", label: __("Sick"), icon: "thermometer" },
	{ kind: "home_office", label: __("Home office"), icon: "home" },
])

const legend = computed(() =>
	["work", "home", "leave", "sick", "holiday", "missing", "open"].map((kind) => ({
		kind,
		...KINDS[kind],
	}))
)

const monthLabel = computed(() => cursor.value.format("MMMM YYYY"))

const team = createResource({
	url: "hr_addon.api.team_calendar.get_team_calendar",
	onSuccess() {
		nextTick(scrollToToday)
	},
})

const members = computed(() => {
	const list = team.data?.members || []
	return [...list.filter((member) => member.is_me), ...list.filter((member) => !member.is_me)]
})

const myDays = computed(() => members.value.find((member) => member.is_me)?.days || [])

const dates = computed(() => {
	const start = cursor.value.startOf("month")
	return Array.from({ length: start.daysInMonth() }, (_, index) => {
		const date = start.date(index + 1)
		return {
			key: date.format("YYYY-MM-DD"),
			day: date.date(),
			weekday: date.format("dd"),
			isWeekend: [0, 6].includes(date.day()),
			isToday: date.isSame(dayjs(), "day"),
		}
	})
})

const todaySummary = computed(() => {
	const today = dayjs().format("YYYY-MM-DD")
	const counts = {}
	for (const member of members.value) {
		const cell = member.days.find((day) => day.date === today)
		if (cell) counts[cell.kind] = (counts[cell.kind] || 0) + 1
	}
	if (!Object.keys(counts).length) return ""
	const away = (counts.leave || 0) + (counts.sick || 0)
	return __("Today: {0} at work, {1} home office, {2} away", [
		(counts.work || 0) + (counts.open || 0),
		counts.home || 0,
		away,
	])
})

const rangeIsSingleDay = computed(() => range.value && range.value.start === range.value.end)
const isFutureRange = computed(() => range.value && dayjs(range.value.start).isAfter(dayjs(), "day"))

const rangeCells = computed(() => {
	if (!range.value) return []
	return myDays.value.filter((cell) => cell.date >= range.value.start && cell.date <= range.value.end)
})

// A single tapped day that already has a request opens that request.
const rangeRequest = computed(() => {
	if (!rangeIsSingleDay.value) return null
	const cell = rangeCells.value[0]
	return cell?.request ? { ...cell.request, pending: cell.pending } : null
})

const rangeTitle = computed(() => {
	if (!range.value) return ""
	const start = dayjs(range.value.start)
	const end = dayjs(range.value.end)
	if (rangeIsSingleDay.value) return start.format("dddd, D MMM")
	const sameMonth = start.isSame(end, "month")
	return `${start.format(sameMonth ? "D" : "D MMM")} – ${end.format("D MMM")}`
})

const rangeHint = computed(() => {
	if (!range.value) return ""
	if (rangeRequest.value) return describe(rangeCells.value[0])
	const total = rangeCells.value.length
	const working = rangeCells.value.filter((cell) => !["free", "holiday"].includes(cell.kind)).length
	const days = total === 1 ? __("1 day") : __("{0} days", [total])
	if (total === 1) return rangeIsSingleDay.value ? describe(rangeCells.value[0]) : days
	return __("{0} · {1} working days · tap a day to start over", [days, working])
})

function load() {
	selected.value = null
	team.submit({
		year: cursor.value.year(),
		month: cursor.value.month() + 1,
		department: department.value || null,
	})
}

function shiftMonth(offset) {
	cursor.value = cursor.value.add(offset, "month")
}

function scrollToToday() {
	const container = scroller.value
	const column = container?.querySelector("[data-today]")
	if (!container || !column) return
	container.scrollLeft = Math.max(0, column.offsetLeft - container.clientWidth / 2)
}

function cellClass(cell) {
	const tone = (KINDS[cell.kind] || KINDS.free).tone
	if (!cell.pending) return tone
	return `${tone} border-2 border-dashed ${PENDING_BORDERS[cell.kind] || "border-gray-400"}`
}

function icon(cell) {
	return (KINDS[cell.kind] || KINDS.free).icon
}

function describe(cell) {
	if (!cell) return ""
	const hours = cell.hours ? ` · ${Number(cell.hours).toFixed(2)} h` : ""
	const pending = cell.pending ? ` (${__("waiting for approval")})` : ""
	const half = cell.half_day ? ` · ${__("half day")}` : ""
	switch (cell.kind) {
		case "work":
			return `${__("At work")}${hours}`
		case "home":
			return `${__("Home office")}${half}${pending}${hours}`
		case "leave":
			return `${__("Vacation")} · ${cell.leave_type || ""}${half}${pending}`
		case "sick":
			return `${__("Sick")} · ${cell.leave_type}${half}${pending}`
		case "holiday":
			return __("Holiday")
		case "missing":
			return __("No check-in or leave recorded")
		case "open":
			return __("Planned workday")
		default:
			return hours ? `${__("Free day")}${hours}` : __("Free day")
	}
}

function select(member, cell) {
	selected.value = { member, cell }
}

function isSelected(member, cell) {
	return selected.value?.member.employee === member.employee && selected.value?.cell.date === cell.date
}

function inRange(date) {
	return Boolean(range.value) && date >= range.value.start && date <= range.value.end
}

let dragAnchor = null
let dragMoved = false

function setRange(a, b) {
	range.value = a <= b ? { start: a, end: b } : { start: b, end: a }
}

// Tap one day, then tap the last day of the period. Tapping the only selected day clears it.
function tapMyDay(cell) {
	selected.value = null
	if (dragMoved) {
		dragMoved = false
		return
	}
	const current = range.value
	if (current && current.start === current.end) {
		if (cell.date === current.start) clearRange()
		else setRange(current.start, cell.date)
		return
	}
	setRange(cell.date, cell.date)
}

// Mouse only: on touch screens a drag has to scroll the table.
function startDrag(cell, event) {
	if (event.pointerType !== "mouse") return
	dragAnchor = cell.date
	dragMoved = false
	dragging.value = true
}

function dragRange(cell) {
	if (!dragging.value || cell.date === dragAnchor) return
	dragMoved = true
	selected.value = null
	setRange(dragAnchor, cell.date)
}

function stopDrag() {
	dragging.value = false
	// The click after a drag lands on the table, not on a day, so drop the flag afterwards.
	setTimeout(() => (dragMoved = false), 0)
}

function clearRange() {
	range.value = null
	dragging.value = false
}

function plan(kind) {
	sheet.request = null
	sheet.kind = kind
	sheet.open = true
}

function openExisting() {
	sheet.request = { kind: rangeRequest.value.kind, name: rangeRequest.value.name }
	sheet.open = true
}

function onRequestChanged(request, message) {
	sheet.open = false
	clearRange()
	toast({
		title: message,
		icon: "check-circle",
		position: "bottom-center",
		iconClasses: "text-green-500",
	})
	load()
}

function openMyDay(date) {
	router.push({ name: "WorkdayDayView", params: { date } })
}

onMounted(() => window.addEventListener("pointerup", stopDrag))
onBeforeUnmount(() => window.removeEventListener("pointerup", stopDrag))

watch([cursor, department], () => {
	clearRange()
	load()
}, { immediate: true })

onIonViewWillEnter(() => {
	if (team.data) load()
})
</script>
