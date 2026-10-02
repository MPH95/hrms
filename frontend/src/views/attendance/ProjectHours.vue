<template>
	<ion-page>
		<ion-header class="ion-no-border app-shell-header">
			<div class="w-full app-shell">
				<div class="flex flex-row bg-white shadow-sm py-4 px-3 items-center justify-between border-b">
					<div class="flex flex-row items-center">
						<Button variant="ghost" class="!px-1 mr-1 hover:bg-white" @click="router.back()">
							<FeatherIcon name="chevron-left" class="h-5 w-5" />
						</Button>
						<h2 class="text-xl font-semibold text-gray-900">{{ __("Project hours") }}</h2>
					</div>
					<div class="flex flex-row items-center gap-2">
						<Button variant="ghost" class="!px-2" @click="shiftMonth(-1)">
							<FeatherIcon name="chevron-left" class="h-5 w-5" />
						</Button>
						<button type="button" class="text-sm font-medium text-gray-800 min-w-28 text-center" @click="backToThisMonth">
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
			<div class="flex flex-col min-h-full w-full app-shell p-4 gap-4">
				<p v-if="openDayCount" class="text-sm text-gray-600 px-1">
					{{
						openDayCount === 1
							? __("1 day still has check-in hours to book")
							: __("{0} days still have check-in hours to book", [openDayCount])
					}}
				</p>

				<div class="flex flex-col gap-2 sm:hidden">
					<button
						v-for="day in monthDays"
						:key="day.date"
						type="button"
						class="flex items-center justify-between gap-3 rounded-xl px-4 py-3 text-left border"
						:class="dayButtonClass(day.date)"
						@click="selectDay(day.date)"
					>
						<div class="min-w-0">
							<div class="text-base font-medium">{{ dayjs(day.date).format("ddd, D MMM") }}</div>
							<div v-if="checkinOpen(day) > 0" class="text-xs mt-0.5" :class="day.date === selectedDate ? 'text-amber-200' : 'text-amber-800'">
								{{ __("{0} open", [formatHours(checkinOpen(day))]) }}
							</div>
							<div v-else-if="Number(day.worked_hours) > 0" class="text-xs mt-0.5 opacity-70">
								{{ __("Check-in hours are booked") }}
							</div>
						</div>
						<div v-if="bookedOn(day.date)" class="text-sm font-semibold shrink-0">
							{{ formatHours(bookedOn(day.date)) }}
						</div>
					</button>
				</div>

				<div class="hidden sm:block bg-white rounded-2xl p-3 sm:p-5">
					<div class="grid grid-cols-7 gap-2 mb-2">
						<div v-for="label in weekdayLabels" :key="label" class="text-center text-xs font-medium text-gray-500 py-1">
							{{ label }}
						</div>
					</div>
					<div class="grid grid-cols-7 gap-2">
						<div v-for="(cell, index) in calendarCells" :key="cell?.date || `blank-${index}`">
							<button
								v-if="cell"
								type="button"
								class="w-full min-h-16 rounded-xl text-sm font-semibold flex flex-col items-center justify-center gap-0.5"
								:class="dayButtonClass(cell.date)"
								@click="selectDay(cell.date)"
							>
								<span>{{ dayjs(cell.date).date() }}</span>
								<span
									v-if="checkinOpen(cell) > 0"
									class="text-[11px] font-medium leading-none"
									:class="cell.date === selectedDate ? 'text-amber-200' : 'text-amber-700'"
								>
									{{ compactHours(checkinOpen(cell)) }} h
								</span>
								<span
									v-else
									class="h-1.5 w-1.5 rounded-full"
									:class="hasBooking(cell.date) ? 'bg-current' : 'bg-transparent'"
								/>
							</button>
						</div>
					</div>
					<p class="text-xs text-gray-500 mt-3">{{ __("A number on a day is check-in hours that are not booked yet.") }}</p>
				</div>

				<div class="bg-white rounded-2xl p-4 flex flex-col gap-4">
					<div>
						<div class="text-lg font-semibold text-gray-900">{{ selectedLabel }}</div>
						<div v-if="selectedDay?.target_hours" class="text-sm text-gray-500 mt-0.5">
							{{ __("Target {0}", [formatHours(selectedDay.target_hours)]) }}
						</div>
					</div>
					<div class="grid grid-cols-3 gap-2">
						<div class="rounded-xl bg-gray-50 px-3 py-2">
							<div class="text-[11px] text-gray-500">{{ __("Worked") }}</div>
							<div class="text-base font-semibold text-gray-900">{{ formatHours(selectedDay?.worked_hours) }}</div>
						</div>
						<div class="rounded-xl bg-gray-50 px-3 py-2">
							<div class="text-[11px] text-gray-500">{{ __("Booked") }}</div>
							<div class="text-base font-semibold text-gray-900">{{ formatHours(dayBooked) }}</div>
						</div>
						<div class="rounded-xl px-3 py-2" :class="dayOpen < 0 ? 'bg-orange-50' : 'bg-gray-50'">
							<div class="text-[11px]" :class="dayOpen < 0 ? 'text-orange-700' : 'text-gray-500'">
								{{ dayOpen < 0 ? __("Over") : __("Open") }}
							</div>
							<div class="text-base font-semibold" :class="dayOpen < 0 ? 'text-orange-800' : 'text-gray-900'">
								{{ formatHours(Math.abs(dayOpen)) }}
							</div>
						</div>
					</div>

					<div v-if="projectGroups.length" class="flex flex-col gap-3">
						<div v-for="group in projectGroups" :key="group.project" class="border rounded-xl">
							<div class="flex items-center justify-between gap-3 px-3 py-2 border-b">
								<div class="text-sm font-medium text-gray-900 truncate">{{ group.project_name }}</div>
								<div class="text-sm font-semibold text-gray-900 shrink-0">{{ formatHours(group.hours) }}</div>
							</div>
							<div
								v-for="entry in group.entries"
								:key="entry.activity_type"
								class="px-3 py-3 border-b last:border-b-0"
							>
								<div v-if="isEditing(entry)" class="flex flex-col gap-3">
									<div class="text-sm text-gray-700">{{ activityLabel(entry.activity_type) }}</div>
									<div class="flex flex-col gap-2">
										<div class="flex items-center justify-between gap-3">
											<div>
												<div class="text-sm text-gray-500">{{ __("Hours") }}</div>
												<div class="text-xs text-gray-400">{{ __("Up to {0}", [formatHours(editRoom)]) }}</div>
											</div>
											<div class="flex items-center gap-3">
												<Button type="button" variant="outline" class="!px-3" :disabled="editing.hours <= 0.5" @click="stepEdit(-0.5)">
													<FeatherIcon name="minus" class="h-4 w-4" />
												</Button>
												<div class="w-14 text-center text-lg font-semibold">{{ selectorHours(editing.hours) }}</div>
												<Button type="button" variant="outline" class="!px-3" :disabled="editing.hours + 0.001 >= editRoom" @click="stepEdit(0.5)">
													<FeatherIcon name="plus" class="h-4 w-4" />
												</Button>
											</div>
										</div>
										<Button
											type="button"
											variant="outline"
											:disabled="editing.hours + 0.001 >= editRoom"
											@click="editing.hours = editRoom"
										>
											{{ __("All {0}", [formatHours(editRoom)]) }}
										</Button>
									</div>
									<label class="flex flex-col gap-1 text-sm text-gray-700">
										{{ __("What did you do?") }}
										<textarea
											v-model="editing.description"
											rows="3"
											maxlength="500"
											required
											class="rounded border px-3 py-2 text-gray-900 bg-white"
										/>
									</label>
									<div class="flex gap-2">
										<Button variant="solid" class="flex-1" :loading="saving.loading" :disabled="!editing.description.trim()" @click="saveEdit">
											{{ __("Save") }}
										</Button>
										<Button variant="outline" @click="editing = null">{{ __("Cancel") }}</Button>
									</div>
								</div>
								<div v-else class="flex items-center justify-between gap-3">
									<div class="min-w-0">
										<div class="text-sm text-gray-800">{{ activityLabel(entry.activity_type) }}</div>
										<div class="text-xs text-gray-500">{{ formatHours(entry.hours) }}</div>
										<div v-if="entry.description" class="text-sm text-gray-600 mt-1 whitespace-pre-wrap break-words">
											{{ entry.description }}
										</div>
									</div>
									<div class="flex items-center gap-1 shrink-0">
										<Button variant="ghost" class="!px-2" @click="startEdit(entry)">
											<FeatherIcon name="edit-2" class="h-4 w-4" />
										</Button>
										<Button variant="ghost" class="!px-2" :aria-label="__('Remove')" @click="removeEntry(entry)">
											<FeatherIcon name="trash-2" class="h-4 w-4" />
										</Button>
									</div>
								</div>
							</div>
						</div>
					</div>
					<div v-else class="text-sm text-gray-500">{{ __("Nothing booked on this day.") }}</div>

					<form v-if="adding" class="border rounded-xl p-3 flex flex-col gap-3" @submit.prevent="addEntry">
						<label class="flex flex-col gap-1 text-sm text-gray-700">
							{{ __("Project") }}
							<select v-model="draft.project" class="rounded border px-3 py-2 text-gray-900 bg-white" :disabled="!projects.length">
								<option v-if="!projects.length" value="">{{ __("No open projects") }}</option>
								<option v-for="project in projects" :key="project.name" :value="project.name">
									{{ project.project_name }}
								</option>
							</select>
						</label>
						<label class="flex flex-col gap-1 text-sm text-gray-700">
							{{ __("Activity") }}
							<select v-model="draft.activity_type" class="rounded border px-3 py-2 text-gray-900 bg-white">
								<option v-for="activity in activities" :key="activity.name" :value="activity.name">
									{{ activity.label }}
								</option>
							</select>
						</label>
						<div class="flex flex-col gap-2">
							<div class="flex items-center justify-between gap-3">
								<div>
									<div class="text-sm text-gray-700">{{ __("Hours") }}</div>
									<div class="text-xs text-gray-400">{{ __("Up to {0}", [formatHours(addRoom)]) }}</div>
								</div>
								<div class="flex items-center gap-3">
									<Button type="button" variant="outline" class="!px-3" :disabled="draft.hours <= 0.5" @click="stepDraft(-0.5)">
										<FeatherIcon name="minus" class="h-4 w-4" />
									</Button>
									<div class="w-14 text-center text-lg font-semibold">{{ selectorHours(draft.hours) }}</div>
									<Button type="button" variant="outline" class="!px-3" :disabled="draft.hours + 0.001 >= addRoom" @click="stepDraft(0.5)">
										<FeatherIcon name="plus" class="h-4 w-4" />
									</Button>
								</div>
							</div>
							<Button
								type="button"
								variant="outline"
								:disabled="draft.hours + 0.001 >= addRoom"
								@click="draft.hours = addRoom"
							>
								{{ __("All {0}", [formatHours(addRoom)]) }}
							</Button>
						</div>
						<label class="flex flex-col gap-1 text-sm text-gray-700">
							{{ __("What did you do?") }}
							<textarea
								v-model="draft.description"
								rows="3"
								maxlength="500"
								required
								class="rounded border px-3 py-2 text-gray-900 bg-white"
							/>
						</label>
						<div class="flex gap-2">
							<Button variant="solid" class="flex-1" type="submit" :loading="saving.loading" :disabled="!canAdd">
								{{ __("Save") }}
							</Button>
							<Button variant="outline" type="button" @click="adding = false">{{ __("Cancel") }}</Button>
						</div>
					</form>
					<Button v-else variant="solid" :disabled="!canStartAdd" @click="openAdd">
						{{ __("Add hours") }}
					</Button>
					<p v-if="!adding && !editing && projects.length && bookableOpen < 0.01" class="text-sm text-gray-500">
						{{
							Number(selectedDay?.worked_hours) > 0
								? __("No open hours left on this day.")
								: __("Hours can be booked once there is a check-in.")
						}}
					</p>
					<p v-if="errorMessage" class="text-sm text-red-700">{{ errorMessage }}</p>
				</div>

				<div class="bg-white rounded-2xl p-4 flex flex-col gap-4">
					<div>
						<div class="text-lg font-semibold text-gray-900">{{ __("This month") }}</div>
						<div class="text-sm text-gray-500 mt-0.5">{{ monthLabel }}</div>
					</div>
					<div class="grid grid-cols-3 gap-2">
						<div class="rounded-xl bg-gray-50 px-3 py-2">
							<div class="text-[11px] text-gray-500">{{ __("Worked") }}</div>
							<div class="text-base font-semibold text-gray-900">{{ formatHours(monthWorked) }}</div>
						</div>
						<div class="rounded-xl bg-gray-50 px-3 py-2">
							<div class="text-[11px] text-gray-500">{{ __("Booked") }}</div>
							<div class="text-base font-semibold text-gray-900">{{ formatHours(monthBooked) }}</div>
						</div>
						<div class="rounded-xl px-3 py-2" :class="monthStillOpen > 0 ? 'bg-amber-50' : 'bg-gray-50'">
							<div class="text-[11px]" :class="monthStillOpen > 0 ? 'text-amber-800' : 'text-gray-500'">{{ __("Still open") }}</div>
							<div class="text-base font-semibold" :class="monthStillOpen > 0 ? 'text-amber-900' : 'text-gray-900'">
								{{ formatHours(monthStillOpen) }}
							</div>
						</div>
					</div>
					<p class="text-sm text-gray-500">
						{{
							daysWithCheckins === 1
								? __("Checked in on 1 day.")
								: __("Checked in on {0} days.", [daysWithCheckins])
						}}
						<span v-if="openDayCount">
							{{
								openDayCount === 1
									? __("1 of them still has hours to book.")
									: __("{0} of them still have hours to book.", [openDayCount])
							}}
						</span>
					</p>
					<div v-if="activityTotals.length > 1" class="flex flex-wrap gap-2">
						<div
							v-for="activity in activityTotals"
							:key="activity.name"
							class="rounded-full bg-gray-100 px-3 py-1 text-xs text-gray-700"
						>
							{{ activity.label }} · {{ formatHours(activity.hours) }}
						</div>
					</div>
					<div v-if="monthProjects.length" class="flex flex-col gap-3">
						<div class="text-sm font-medium text-gray-700">{{ __("Projects") }}</div>
						<div v-for="group in monthProjects" :key="group.project" class="border rounded-xl">
							<div class="flex items-center justify-between gap-3 px-3 py-2 border-b">
								<div class="min-w-0">
									<div class="text-sm font-medium text-gray-900 truncate">{{ group.project_name }}</div>
									<div class="text-xs text-gray-500">{{ __("{0} of booked hours", [group.share]) }}</div>
								</div>
								<div class="text-sm font-semibold text-gray-900 shrink-0">{{ formatHours(group.hours) }}</div>
							</div>
							<div
								v-for="activity in group.activities"
								:key="activity.name"
								class="flex items-center justify-between gap-3 px-3 py-2 border-b last:border-b-0"
							>
								<div class="text-sm text-gray-700">{{ activity.label }}</div>
								<div class="text-sm text-gray-900">{{ formatHours(activity.hours) }}</div>
							</div>
						</div>
					</div>
					<p v-else class="text-sm text-gray-500">{{ __("Nothing booked this month.") }}</p>
				</div>
			</div>
		</ion-content>
	</ion-page>
</template>

<script setup>
import { computed, inject, reactive, ref, watch } from "vue"
import { useRouter } from "vue-router"
import { IonPage, IonHeader, IonContent, onIonViewWillEnter } from "@ionic/vue"
import { Button, FeatherIcon, createResource } from "frappe-ui"

const __ = inject("$translate")
const dayjs = inject("$dayjs")
const router = useRouter()

const cursor = ref(dayjs().startOf("month"))
const selectedDate = ref(dayjs().format("YYYY-MM-DD"))
const errorMessage = ref("")
const adding = ref(false)
const editing = ref(null)
const draft = reactive({
	project: "",
	activity_type: "Development",
	hours: 1,
	description: "",
})

const weekdayLabels = [1, 2, 3, 4, 5, 6, 0].map((day) => dayjs().day(day).format("dd"))

const page = createResource({
	url: "hr_addon.api.project_time.get_my_project_time",
	makeParams() {
		return { year: cursor.value.year(), month: cursor.value.month() + 1 }
	},
	auto: true,
	onSuccess() {
		keepSelectionInMonth()
	},
})

const saving = createResource({
	url: "hr_addon.api.project_time.save_my_project_time",
})

const monthLabel = computed(() => cursor.value.format("MMMM YYYY"))
const projects = computed(() => page.data?.projects || [])
const activities = computed(() => page.data?.activities || [])
const monthDays = computed(() => page.data?.days || [])

const calendarCells = computed(() => {
	const leading = (cursor.value.day() + 6) % 7
	const blanks = Array.from({ length: leading }, () => null)
	return blanks.concat(monthDays.value)
})

const selectedDay = computed(() => (page.data?.days || []).find((day) => day.date === selectedDate.value))
const selectedLabel = computed(() => dayjs(selectedDate.value).format("dddd, D MMMM"))

const dayEntries = computed(() => (page.data?.entries || []).filter((entry) => entry.date === selectedDate.value))

const dayBooked = computed(() => dayEntries.value.reduce((sum, entry) => sum + Number(entry.hours || 0), 0))

const dayOpen = computed(() => Number(selectedDay.value?.worked_hours || 0) - dayBooked.value)
const bookableOpen = computed(() => roundHours(Math.max(0, dayOpen.value)))

const projectGroups = computed(() => {
	const groups = []
	for (const entry of dayEntries.value) {
		let group = groups.find((item) => item.project === entry.project)
		if (!group) {
			group = {
				project: entry.project,
				project_name: entry.project_name,
				hours: 0,
				entries: [],
			}
			groups.push(group)
		}
		group.entries.push(entry)
		group.hours += Number(entry.hours || 0)
	}
	return groups
})

const addRoom = computed(() => roomFor(draft.project, draft.activity_type))
const editRoom = computed(() =>
	editing.value ? roomFor(editing.value.project, editing.value.activity_type) : 0
)
const canAdd = computed(
	() =>
		draft.project &&
		draft.activity_type &&
		draft.hours >= 0.01 &&
		draft.hours <= addRoom.value + 0.001 &&
		draft.description.trim() &&
		!saving.loading
)
const canStartAdd = computed(() => projects.value.length && bookableOpen.value >= 0.01)

const monthWorked = computed(() => monthDays.value.reduce((sum, day) => sum + Number(day.worked_hours || 0), 0))
const monthBooked = computed(() =>
	(page.data?.entries || []).reduce((sum, entry) => sum + Number(entry.hours || 0), 0)
)
const daysWithCheckins = computed(() => monthDays.value.filter((day) => Number(day.worked_hours) > 0).length)
const openDayCount = computed(() => monthDays.value.filter((day) => checkinOpen(day) > 0.001).length)
const monthStillOpen = computed(() => monthDays.value.reduce((sum, day) => sum + checkinOpen(day), 0))

const monthProjects = computed(() => {
	const groups = []
	for (const entry of page.data?.entries || []) {
		let group = groups.find((item) => item.project === entry.project)
		if (!group) {
			group = {
				project: entry.project,
				project_name: entry.project_name,
				hours: 0,
				activities: [],
			}
			groups.push(group)
		}
		const hours = Number(entry.hours || 0)
		group.hours += hours
		let activity = group.activities.find((item) => item.name === entry.activity_type)
		if (!activity) {
			activity = { name: entry.activity_type, label: activityLabel(entry.activity_type), hours: 0 }
			group.activities.push(activity)
		}
		activity.hours += hours
	}
	const total = groups.reduce((sum, group) => sum + group.hours, 0)
	for (const group of groups) {
		group.share = total ? `${Math.round((group.hours / total) * 100)}%` : "0%"
		group.activities.sort((a, b) => b.hours - a.hours)
	}
	return groups.sort((a, b) => b.hours - a.hours)
})

const activityTotals = computed(() => {
	const totals = []
	for (const group of monthProjects.value) {
		for (const activity of group.activities) {
			let row = totals.find((item) => item.name === activity.name)
			if (!row) {
				row = { name: activity.name, label: activity.label, hours: 0 }
				totals.push(row)
			}
			row.hours += activity.hours
		}
	}
	return totals.sort((a, b) => b.hours - a.hours)
})

watch(projects, (list) => {
	if (!list.some((project) => project.name === draft.project)) {
		draft.project = list[0]?.name || ""
	}
})

watch(activities, (list) => {
	if (!list.some((activity) => activity.name === draft.activity_type)) {
		draft.activity_type = list[0]?.name || ""
	}
})

watch(addRoom, (room) => {
	if (!adding.value) return
	if (room < 0.01) {
		adding.value = false
		return
	}
	if (draft.hours > room) draft.hours = room
})

onIonViewWillEnter(() => page.reload())

function shiftMonth(delta) {
	cursor.value = cursor.value.add(delta, "month")
	errorMessage.value = ""
	adding.value = false
	editing.value = null
	page.reload()
}

function backToThisMonth() {
	cursor.value = dayjs().startOf("month")
	selectedDate.value = dayjs().format("YYYY-MM-DD")
	page.reload()
}

function keepSelectionInMonth() {
	const days = (page.data?.days || []).map((day) => day.date)
	if (!days.includes(selectedDate.value)) {
		const today = dayjs().format("YYYY-MM-DD")
		selectedDate.value = days.includes(today) ? today : days[0]
	}
}

function selectDay(date) {
	selectedDate.value = date
	adding.value = false
	editing.value = null
	errorMessage.value = ""
}

function bookedOn(date) {
	return (page.data?.entries || [])
		.filter((entry) => entry.date === date)
		.reduce((sum, entry) => sum + Number(entry.hours || 0), 0)
}

function checkinOpen(day) {
	const worked = Number(day?.worked_hours || 0)
	if (worked <= 0) return 0
	return Math.max(0, worked - bookedOn(day.date))
}

function hasBooking(date) {
	return bookedOn(date) > 0
}

function roomFor(project, activity) {
	const existing = dayEntries.value.find((entry) => entry.project === project && entry.activity_type === activity)
	return roundHours(Math.max(0, dayOpen.value + (existing ? Number(existing.hours) : 0)))
}

function roundHours(value) {
	return Math.max(0, Math.round(Number(value || 0) * 100) / 100)
}

function dayButtonClass(date) {
	const day = monthDays.value.find((row) => row.date === date)
	const open = day && checkinOpen(day) > 0.001
	const today = date === dayjs().format("YYYY-MM-DD")
	if (date === selectedDate.value) return "bg-gray-900 text-white border-gray-900"
	if (open && today) return "bg-amber-50 text-gray-900 border-amber-200 ring-1 ring-amber-300"
	if (open) return "bg-amber-50 text-gray-900 border-amber-200"
	if (today) return "bg-gray-100 text-gray-900 border-gray-200 ring-1 ring-gray-300"
	return "bg-white text-gray-800 border-gray-200"
}

function openAdd() {
	const room = roomFor(draft.project, draft.activity_type)
	if (room < 0.01) return
	editing.value = null
	draft.hours = room < 1 ? room : 1
	draft.description = ""
	adding.value = true
}

function startEdit(entry) {
	adding.value = false
	editing.value = {
		project: entry.project,
		activity_type: entry.activity_type,
		hours: Number(entry.hours),
		description: entry.description || "",
	}
}

function isEditing(entry) {
	return (
		editing.value &&
		editing.value.project === entry.project &&
		editing.value.activity_type === entry.activity_type
	)
}

function stepEdit(delta) {
	editing.value.hours = stepWithin(editing.value.hours, delta, editRoom.value)
}

function stepDraft(delta) {
	draft.hours = stepWithin(draft.hours, delta, addRoom.value)
}

function stepWithin(current, delta, max) {
	const cap = roundHours(max)
	const now = roundHours(current)
	if (delta > 0) {
		const stepped = roundHours(Math.round((now + 0.5) * 2) / 2)
		return stepped > cap ? cap : stepped
	}
	if (now <= 0.5) return now
	return Math.max(0.5, roundHours(Math.round((now - 0.5) * 2) / 2))
}

function selectorHours(value) {
	const hours = roundHours(value)
	if (Math.abs(hours * 2 - Math.round(hours * 2)) < 0.001) return hours.toFixed(1)
	return hours.toFixed(2)
}

function compactHours(value) {
	const hours = Math.round(Number(value || 0) * 100) / 100
	if (Math.abs(hours - Math.round(hours)) < 0.001) return String(Math.round(hours))
	if (Math.abs(hours * 2 - Math.round(hours * 2)) < 0.001) return (Math.round(hours * 2) / 2).toFixed(1)
	return hours.toFixed(2)
}

function formatHours(value) {
	return `${Number(value || 0).toFixed(2)} h`
}

function activityLabel(name) {
	return activities.value.find((activity) => activity.name === name)?.label || name
}

async function addEntry() {
	const entries = (page.data?.entries || []).map(plainEntry)
	const existing = entries.find(
		(entry) =>
			entry.date === selectedDate.value &&
			entry.project === draft.project &&
			entry.activity_type === draft.activity_type
	)
	const note = draft.description.trim()
	if (existing) {
		existing.hours = draft.hours
		existing.description = note
	} else {
		entries.push({
			date: selectedDate.value,
			project: draft.project,
			activity_type: draft.activity_type,
			hours: draft.hours,
			description: note,
		})
	}
	if (await persist(entries)) adding.value = false
}

async function saveEdit() {
	const entries = (page.data?.entries || []).map(plainEntry)
	const existing = entries.find(
		(entry) =>
			entry.date === selectedDate.value &&
			entry.project === editing.value.project &&
			entry.activity_type === editing.value.activity_type
	)
	if (existing) {
		existing.hours = editing.value.hours
		existing.description = (editing.value.description || "").trim()
	}
	if (await persist(entries)) editing.value = null
}

async function removeEntry(entry) {
	const entries = (page.data?.entries || [])
		.filter(
			(row) =>
				!(
					row.date === entry.date &&
					row.project === entry.project &&
					row.activity_type === entry.activity_type
				)
		)
		.map(plainEntry)
	editing.value = null
	await persist(entries)
}

function plainEntry(entry) {
	return {
		date: entry.date,
		project: entry.project,
		activity_type: entry.activity_type,
		hours: entry.hours,
		description: entry.description || "",
	}
}

async function persist(entries) {
	errorMessage.value = ""
	try {
		const saved = await saving.submit({
			year: cursor.value.year(),
			month: cursor.value.month() + 1,
			entries,
		})
		page.setData(saved)
		return true
	} catch (error) {
		errorMessage.value = error?.messages?.[0] || error?.message || __("The hours could not be saved.")
		return false
	}
}
</script>
