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
							{{ Number(day.data.actual_working_hours || 0).toFixed(2) }} h
						</div>
					</div>
				</div>
			</div>
		</ion-header>
		<ion-content class="app-shell-content">
			<div class="flex flex-col min-h-full w-full app-shell p-4 gap-5">
				<div v-if="suggestion" class="rounded bg-orange-50 text-orange-800 text-sm px-3 py-2">
					{{ suggestion.reason }}
				</div>
				<div
					v-else-if="day.data?.needs_attention"
					class="rounded bg-orange-50 text-orange-800 text-sm px-3 py-2"
				>
					{{ __("This day is missing a check-in or check-out. Add the punch you forgot, or correct the time.") }}
				</div>

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
				<div v-else class="bg-white rounded p-4">
					<div class="text-base font-semibold text-gray-900 mb-3">{{ formTitle }}</div>
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
			</div>
		</ion-content>
	</ion-page>
</template>

<script setup>
import { computed, inject, ref, watch } from "vue"
import { useRouter } from "vue-router"
import { IonPage, IonHeader, IonContent } from "@ionic/vue"
import { createResource, FeatherIcon } from "frappe-ui"

import CheckinForm from "@/components/CheckinForm.vue"

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

const day = createResource({
	url: "hr_addon.api.employee_app.get_my_day",
})

const removeCheckin = createResource({
	url: "hr_addon.api.employee_app.delete_my_checkin",
})

const suggestion = computed(() => (ignoreSuggestion.value ? null : day.data?.suggestion || null))

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
	if (suggestion.value?.log_type === "OUT") return __("Suggested check-out")
	if (suggestion.value?.log_type === "IN") return __("Suggested check-in")
	return __("Add missing check-in")
})

watch(
	() => props.date,
	(date) => {
		editing.value = null
		ignoreSuggestion.value = false
		removeError.value = ""
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

function onSaved() {
	editing.value = null
	ignoreSuggestion.value = false
	removeError.value = ""
	day.submit({ date: props.date })
}

async function removePunch() {
	removeError.value = ""
	try {
		await removeCheckin.submit({ name: suggestion.value?.name })
		onSaved()
	} catch (err) {
		removeError.value =
			removeCheckin.error?.messages?.[0] || err?.message || __("Could not remove the check-in")
	}
}
</script>
