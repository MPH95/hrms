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
						<div class="text-sm font-medium text-gray-800 min-w-28 text-center">
							{{ monthLabel }}
						</div>
						<Button variant="ghost" class="!px-2" :disabled="isCurrentMonth" @click="shiftMonth(1)">
							<FeatherIcon name="chevron-right" class="h-5 w-5" />
						</Button>
					</div>
				</div>
			</div>
		</ion-header>
		<ion-content class="app-shell-content">
			<div class="flex flex-col min-h-full w-full app-shell p-4 gap-3">
				<div
					v-if="workdays.data?.missing_days"
					class="rounded bg-orange-50 text-orange-800 text-sm px-3 py-2"
				>
					{{ __("{0} days need a check-in correction", [workdays.data.missing_days]) }}
				</div>
				<div v-if="workdays.loading" class="text-sm text-gray-500">{{ __("Loading") }}</div>
				<div v-else-if="!workdays.data?.days?.length" class="text-sm text-gray-500">
					{{ __("No working days in this month") }}
				</div>
				<button
					v-for="day in workdays.data?.days || []"
					:key="day.date"
					type="button"
					class="flex items-center justify-between gap-3 bg-white rounded px-4 py-3 text-left border"
					:class="day.needs_attention ? 'border-orange-300' : 'border-transparent'"
					@click="router.push({ name: 'WorkdayDayView', params: { date: day.date } })"
				>
					<div>
						<div class="text-base font-medium text-gray-900">
							{{ dayjs(day.date).format("ddd, D MMM") }}
						</div>
						<div class="text-xs mt-1" :class="day.needs_attention ? 'text-orange-700' : 'text-gray-500'">
							{{ __(day.status) }}
						</div>
					</div>
					<div class="text-right">
						<div class="text-sm font-semibold text-gray-900">
							{{ Number(day.actual_working_hours || 0).toFixed(2) }} h
						</div>
						<div class="text-xs text-gray-500">
							{{ __("Target {0}", [`${Number(day.target_hours || 0).toFixed(2)} h`]) }}
						</div>
					</div>
				</button>
			</div>
		</ion-content>
	</ion-page>
</template>

<script setup>
import { computed, inject, ref, watch } from "vue"
import { useRouter } from "vue-router"
import { IonPage, IonHeader, IonContent } from "@ionic/vue"
import { createResource, FeatherIcon } from "frappe-ui"

const __ = inject("$translate")
const dayjs = inject("$dayjs")
const router = useRouter()

const cursor = ref(dayjs().startOf("month"))

const monthLabel = computed(() => cursor.value.format("MMMM YYYY"))
const isCurrentMonth = computed(() => cursor.value.isSame(dayjs(), "month"))

const workdays = createResource({
	url: "hr_addon.api.employee_app.get_my_workdays",
})

function load() {
	workdays.submit({
		year: cursor.value.year(),
		month: cursor.value.month() + 1,
	})
}

function shiftMonth(offset) {
	const next = cursor.value.add(offset, "month")
	if (next.isAfter(dayjs(), "month")) return
	cursor.value = next
}

watch(cursor, load, { immediate: true })
</script>
