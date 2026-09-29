<template>
	<BaseLayout>
		<template #body>
			<div class="grid grid-cols-1 lg:grid-cols-2 lg:items-start my-7 p-4 lg:px-8 gap-7 lg:gap-x-10">
				<div class="flex flex-col items-center gap-7 min-w-0">
					<CheckInPanel />
					<HoursSummary />
				</div>
				<div class="flex flex-col items-center gap-7 min-w-0">
					<QuickLinks :items="quickLinks" :title="__('Quick Links')" />
					<RequestPanel />
				</div>
			</div>
		</template>
	</BaseLayout>
</template>

<script setup>
import { inject, markRaw } from "vue"

import CheckInPanel from "@/components/CheckInPanel.vue"
import HoursSummary from "@/components/HoursSummary.vue"
import QuickLinks from "@/components/QuickLinks.vue"
import BaseLayout from "@/components/BaseLayout.vue"
import RequestPanel from "@/components/RequestPanel.vue"
import AttendanceIcon from "@/components/icons/AttendanceIcon.vue"
import ShiftIcon from "@/components/icons/ShiftIcon.vue"
import LeaveIcon from "@/components/icons/LeaveIcon.vue"
import ExpenseIcon from "@/components/icons/ExpenseIcon.vue"
import EmployeeAdvanceIcon from "@/components/icons/EmployeeAdvanceIcon.vue"
import SalaryIcon from "@/components/icons/SalaryIcon.vue"
import TeamIcon from "@/components/icons/TeamIcon.vue"

const __ = inject("$translate")

const quickLinks = [
	{
		icon: markRaw(TeamIcon),
		title: __("Team Calendar"),
		route: "TeamCalendarView",
	},
	{
		icon: markRaw(AttendanceIcon),
		title: __("Request Attendance"),
		route: "AttendanceRequestFormView",
	},
	{
		icon: markRaw(ShiftIcon),
		title: __("Request a Shift"),
		route: "ShiftRequestFormView",
	},
	{
		icon: markRaw(LeaveIcon),
		title: __("Request Leave"),
		route: "LeaveApplicationFormView",
	},
	{
		icon: markRaw(ExpenseIcon),
		title: __("Claim an Expense"),
		route: "ExpenseClaimFormView",
	},
	{
		icon: markRaw(EmployeeAdvanceIcon),
		title: __("Request an Advance"),
		route: "EmployeeAdvanceFormView",
	},
	{
		icon: markRaw(SalaryIcon),
		title: __("View Salary Slips"),
		route: "SalarySlipsDashboard",
	},
]
</script>
