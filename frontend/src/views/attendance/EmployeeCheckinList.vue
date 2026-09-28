<template>
	<ion-page>
		<ListView
			ref="listView"
			doctype="Employee Checkin"
			:pageTitle="__('Employee Checkin History')"
			:fields="EMPLOYEE_CHECKIN_FIELDS"
			:filterConfig="FILTER_CONFIG"
			:onCheckinClick="openEditor"
		>
			<template #actions>
				<Button variant="solid" class="mr-2" @click="openEditor(null)">
					{{ __("Add") }}
				</Button>
			</template>
		</ListView>

		<CustomIonModal :isOpen="editorOpen" @did-dismiss="closeEditor">
			<template #actionSheet>
				<div class="bg-white w-full flex flex-col p-4 pb-8 gap-4">
					<div class="text-lg font-bold text-gray-900 text-center pt-4">
						{{ editing?.name ? __("Correct check-in") : __("Add missing check-in") }}
					</div>
					<CheckinForm :checkin="editing" @saved="onSaved" />
				</div>
			</template>
		</CustomIonModal>
	</ion-page>
</template>

<script setup>
import { inject, ref } from "vue"
import { IonPage } from "@ionic/vue"

import ListView from "@/components/ListView.vue"
import CustomIonModal from "@/components/CustomIonModal.vue"
import CheckinForm from "@/components/CheckinForm.vue"

const __ = inject("$translate")

const EMPLOYEE_CHECKIN_FIELDS = ["name", "log_type", "time", "latitude", "longitude", "attendance"]

const FILTER_CONFIG = [
	{
		fieldname: "log_type",
		fieldtype: "Select",
		label: __("Log Type"),
		options: "IN\nOUT",
	},
	{
		fieldname: "shift",
		fieldtype: "Link",
		label: __("Shift"),
		options: "Shift Type",
	},
]

const listView = ref(null)
const editorOpen = ref(false)
const editing = ref(null)

function openEditor(checkin) {
	editing.value = checkin ? { ...checkin } : null
	editorOpen.value = true
}

function closeEditor() {
	editorOpen.value = false
	editing.value = null
}

function onSaved() {
	closeEditor()
	listView.value?.reload()
}
</script>
