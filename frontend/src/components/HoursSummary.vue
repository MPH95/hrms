<template>
	<button
		type="button"
		class="flex flex-col gap-4 w-full bg-white rounded p-4 text-left"
		@click="router.push({ name: 'WorkdayListView' })"
	>
		<div class="flex flex-col gap-1 w-full">
			<div class="text-sm font-medium text-gray-500">{{ __("Today") }}</div>
			<div class="text-2xl font-bold text-gray-900">
				{{ formatHours(summary.data?.today_actual) }}
				<span v-if="summary.data?.today_open" class="text-sm font-medium text-gray-500">
					{{ __("so far") }}
				</span>
				<span v-else-if="!Number(summary.data?.today_actual)" class="text-sm font-medium text-gray-500">
					{{ __("not checked in") }}
				</span>
			</div>
			<div class="text-sm text-gray-500">
				{{ __("Target {0}", [formatHours(summary.data?.today_target)]) }}
			</div>
		</div>
		<div class="flex items-center justify-between gap-3 w-full border-t pt-3">
			<div class="text-sm text-gray-500">{{ __("Overtime balance") }}</div>
			<div
				v-if="summary.data?.overtime_enabled"
				class="text-base font-semibold"
				:class="balanceClass(summary.data.overtime_balance)"
			>
				{{ formatBalance(summary.data.overtime_balance) }}
			</div>
			<div v-else class="text-sm text-gray-500">{{ __("Not enabled") }}</div>
		</div>
		<div class="flex items-start justify-between gap-3 w-full border-t pt-3">
			<div>
				<div class="text-sm font-medium text-gray-500">{{ monthLabel }}</div>
				<div class="text-2xl font-bold text-gray-900 mt-1">
					{{ formatHours(summary.data?.month_actual) }}
				</div>
				<div class="text-sm text-gray-500 mt-1">
					{{ __("Target {0}", [formatHours(summary.data?.month_target)]) }}
					<span class="whitespace-pre"> &middot; </span>
					<span :class="balanceClass(summary.data?.month_balance)">
						{{ formatBalance(summary.data?.month_balance) }}
					</span>
				</div>
			</div>
			<div
				v-if="summary.data?.missing_days"
				class="shrink-0 rounded-full bg-orange-100 text-orange-800 text-xs font-semibold px-2.5 py-1"
			>
				{{ __("{0} to fix", [summary.data.missing_days]) }}
			</div>
		</div>
		<div class="flex flex-col gap-2 w-full border-t pt-3">
			<div class="text-sm text-gray-500">{{ __("This year") }}</div>
			<div class="flex items-start justify-between gap-3 w-full">
				<div>
					<div class="text-xs text-gray-500">{{ __("Actual") }}</div>
					<div class="text-base font-semibold text-gray-900">
						{{ formatHours(summary.data?.year_actual) }}
					</div>
				</div>
				<div class="text-right">
					<div class="text-xs text-gray-500">{{ __("Target") }}</div>
					<div class="text-base font-semibold text-gray-900">
						{{ formatHours(summary.data?.year_target) }}
					</div>
				</div>
			</div>
		</div>
		<div class="flex items-center justify-between gap-3 w-full border-t pt-3">
			<div class="flex items-center gap-2 text-sm font-medium text-gray-900">
				<FeatherIcon name="calendar" class="h-4 w-4" />
				{{ __("Open workday calendar") }}
			</div>
			<div class="flex items-center gap-1 text-xs text-gray-500">
				<span class="hidden sm:inline">{{ __("Fix days, plan vacation & home office") }}</span>
				<FeatherIcon name="chevron-right" class="h-4 w-4" />
			</div>
		</div>
	</button>
</template>

<script setup>
import { computed, inject } from "vue"
import { useRouter } from "vue-router"
import { createResource, FeatherIcon } from "frappe-ui"

const __ = inject("$translate")
const dayjs = inject("$dayjs")
const router = useRouter()

const summary = createResource({
	url: "hr_addon.api.employee_app.get_my_hours_summary",
	auto: true,
})

function reload() {
	summary.reload()
}

defineExpose({ reload })

const monthLabel = computed(() => dayjs().format("MMMM YYYY"))

function formatHours(value) {
	return `${Number(value || 0).toFixed(2)} h`
}

function formatBalance(value) {
	const hours = Number(value || 0)
	const sign = hours > 0 ? "+" : ""
	return `${sign}${hours.toFixed(2)} h`
}

function balanceClass(value) {
	const hours = Number(value || 0)
	if (hours < 0) return "text-orange-700"
	if (hours > 0) return "text-green-700"
	return "text-gray-500"
}
</script>
