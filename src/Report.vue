<template>
	<div class="app-start-box" v-if="isAppLoading">
		<v-progress-circular indeterminate />
	</div>

	<template v-else>
		<div class="container">
			<div class="filters-box">
				<v-select
					label="Тип отчёта"
					hint="Выберите: по отделам или по сотрудникам"
					persistent-hint
					:items="modeItems"
					v-model="reportMode"
				/>

				<v-select
					v-model="selectedDepartment"
					:items="departmentItems"
					item-title="title"
					item-value="value"
					label="Отдел"
					hint="Выберите нужный отдел"
					persistent-hint
					dense
				>
					<template #item="{ item, props }">
						<div
							v-bind="props"
							class="select-item d-flex justify-space-between align-center"
							:class="{ 'text-disabled': item.disabled }"
						>
							<span>{{ item.title }}</span>

							<span v-if="item.count" class="text-caption grey--text">{{
								item.count
							}}</span>
						</div>
					</template>
				</v-select>

				<v-date-input
					label="Дата с"
					hint="Дата создания сделки"
					persistent-hint
					v-model="dateFrom"
					clearable
				/>

				<v-date-input
					label="Дата по"
					hint="Дата создания сделки"
					persistent-hint
					v-model="dateTo"
					clearable
				/>
			</div>

			<div class="filter-title">Статусы сделок</div>
			<div class="filter-subtitle">
				Выберите, какие сделки учитывать в отчёте
			</div>

			<div class="filter-options" style="display: flex; gap: 1rem">
				<v-checkbox
					label="Все статусы"
					color="primary"
					value="ALL"
					v-model="statusFilter"
					@update:modelValue="toggleAllStatus"
				/>

				<v-checkbox
					v-for="s in statusOptions"
					:key="s.value"
					color="primary"
					:label="s.label"
					:value="s.value"
					v-model="statusFilter"
					:disabled="statusFilter.includes('ALL')"
				/>
			</div>

			<v-btn
				color="primary"
				@click="loadReport"
				:disabled="isLoading"
				:loading="isLoading"
			>
				Сформировать отчёт
			</v-btn>
		</div>

		<div class="content-box">
			<!-- Когда ещё не строили отчёт -->
			<v-alert v-if="!isReportBuilt" type="info" variant="tonal" class="mt-4">
				Отчёт пока не сформирован. Выберите фильтры и нажмите «Обновить данные»
			</v-alert>

			<!-- Когда отчёт построен, но данных нет -->
			<v-alert
				v-else-if="isReportBuilt && !rows.length"
				type="warning"
				variant="tonal"
				class="mt-4"
			>
				Нет данных по выбранным фильтрам. Попробуйте изменить параметры отчёта
			</v-alert>

			<!-- BI-информация о том, что показываются только активные элементы -->
			<div v-if="isReportBuilt && rows.length" class="table-info">
				🔹 В отчёт включены только отделы или сотрудники, у которых есть сделки
				за выбранный период
			</div>

			<v-table v-if="rows.length" density="default">
				<thead>
					<tr>
						<th v-for="col in columns" :key="col">{{ labels[col] }}</th>
					</tr>
				</thead>
				<tbody>
					<tr v-for="row in rows" :key="row.key" class="row">
						<td v-for="col in columns" :key="col">
							{{ row[col] }}
						</td>
					</tr>
				</tbody>
			</v-table>
		</div>
	</template>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue'

type User = {
	ID: string
	NAME: string
	LAST_NAME: string
	UF_DEPARTMENT: number[]
}
type Deal = {
	ID: string
	ASSIGNED_BY_ID: string
	STAGE_SEMANTIC_ID: 'P' | 'S' | 'F'
	DATE_CREATE: string
}
type Department = { ID: number; NAME: string }

const sleep = (ms: number) => new Promise(res => setTimeout(res, ms))

const isAppLoading = ref(true)
/* ================= STATE ================= */
const users = ref<User[]>([])
const deals = ref<Deal[]>([])
const departments = ref<Department[]>([])

const isReportBuilt = ref(false)

const selectedDepartment = ref<number | null>(null)
const reportMode = ref<'departments' | 'users'>('departments')

const dateFrom = ref<string | null>(null)
const dateTo = ref<string | null>(null)

const statusFilter = ref<string[]>(['ALL'])

const rows = ref<any[]>([])
const columns = ref<string[]>([])

const isLoading = ref<boolean>(false)

/* ================= UI ================= */
const modeItems = [
	{ title: 'По отделам', value: 'departments' },
	{ title: 'По сотрудникам', value: 'users' },
]

const statusOptions = [
	{ label: 'В работе', value: 'P' },
	{ label: 'Успешные', value: 'S' },
	{ label: 'Проваленные', value: 'F' },
]

const labels: Record<string, string> = {
	departmentName: 'Отдел',
	userName: 'Сотрудник',
	total: 'Всего',
	inWork: 'В работе',
	success: 'Успешные',
	fail: 'Проваленные',
}

/* ================= HELPERS ================= */
function toggleAllStatus() {
	if (statusFilter.value.includes('ALL')) statusFilter.value = ['ALL']
}

/* ================= API ================= */
function loadUsers(): Promise<User[]> {
	return new Promise(resolve =>
		BX24.callMethod('user.get', {}, (res: any) => resolve(res.data())),
	)
}

function loadDepartments(): Promise<Department[]> {
	return new Promise(resolve =>
		BX24.callMethod('department.get', {}, (res: any) => resolve(res.data())),
	)
}

function loadDeals(): Promise<Deal[]> {
	const filter: Record<string, any> = {}
	if (dateFrom.value) filter['>=DATE_CREATE'] = dateFrom.value
	if (dateTo.value) filter['<=DATE_CREATE'] = dateTo.value
	if (!statusFilter.value.includes('ALL'))
		filter['STAGE_SEMANTIC_ID'] = statusFilter.value

	return new Promise(resolve =>
		BX24.callMethod(
			'crm.deal.list',
			{
				select: ['ID', 'ASSIGNED_BY_ID', 'STAGE_SEMANTIC_ID', 'DATE_CREATE'],
				filter,
				start: -1,
			},
			(res: any) => resolve(res.data()),
		),
	)
}

/* ================= DYNAMIC DEPARTMENT SELECT ================= */
const departmentItems = ref<{ title: string; value: number | null }[]>([
	{ title: 'Все', value: null },
])

function updateDepartmentItems() {
	// 1️⃣ Счётчик сделок по отделам
	const departmentDealCount = new Map<number, number>()

	deals.value.forEach(deal => {
		const user = users.value.find(u => u.ID === deal.ASSIGNED_BY_ID)
		if (!user) return

		const deps = Array.isArray(user.UF_DEPARTMENT)
			? user.UF_DEPARTMENT
			: user.UF_DEPARTMENT
				? [user.UF_DEPARTMENT]
				: []

		deps.forEach(depId => {
			const id = Number(depId)
			if (!isNaN(id)) {
				departmentDealCount.set(id, (departmentDealCount.get(id) || 0) + 1)
			}
		})
	})

	// 2️⃣ Активные отделы (сделки > 0)
	const activeDepartments = departments.value
		.filter(d => (departmentDealCount.get(Number(d.ID)) || 0) > 0)
		.map(d => ({
			title: d.NAME,
			value: d.ID,
			count: departmentDealCount.get(Number(d.ID)),
			disabled: false,
		}))

	// 3️⃣ Пустые отделы (сделки = 0)
	const emptyDepartments = departments.value
		.filter(d => (departmentDealCount.get(Number(d.ID)) || 0) === 0)
		.map(d => ({
			title: d.NAME,
			value: d.ID,
			count: 0,
			disabled: true,
		}))

	// 4️⃣ Пункт "Все" с суммарным количеством
	const totalCount = Array.from(departmentDealCount.values()).reduce(
		(a, b) => a + b,
		0,
	)
	const allItem = {
		title: `Все`,
		value: null,
		count: totalCount,
		disabled: false,
	}

	// 5️⃣ Обновляем departmentItems
	departmentItems.value = [allItem, ...activeDepartments, ...emptyDepartments]
}
/* ================= MAIN ================= */
async function loadReport() {
	isLoading.value = true

	users.value = await loadUsers()
	departments.value = await loadDepartments()
	deals.value = await loadDeals()

	// Обновляем список отделов в селекте
	updateDepartmentItems()

	// Создаём быстрый доступ к пользователям и отделам по ID
	const userMap: Record<string, User> = Object.fromEntries(
		users.value.map(u => [u.ID, u]),
	)
	const departmentMap: Record<number, string> = Object.fromEntries(
		departments.value.map(d => [d.ID, d.NAME]),
	)

	const map: Record<string, any> = {}

	for (const deal of deals.value) {
		const user = userMap[deal.ASSIGNED_BY_ID]
		if (!user) continue

		// --- Режим: по отделам ---
		if (reportMode.value === 'departments') {
			for (const depId of user.UF_DEPARTMENT || []) {
				// фильтруем по выбранному отделу
				if (
					selectedDepartment.value &&
					Number(depId) != selectedDepartment.value
				)
					continue

				if (!map[depId]) {
					map[depId] = {
						key: depId,
						departmentName: departmentMap[depId] || `Отдел ${depId}`,
						total: 0,
						inWork: 0,
						success: 0,
						fail: 0,
					}
				}

				map[depId].total++
				if (deal.STAGE_SEMANTIC_ID === 'P') map[depId].inWork++
				if (deal.STAGE_SEMANTIC_ID === 'S') map[depId].success++
				if (deal.STAGE_SEMANTIC_ID === 'F') map[depId].fail++
			}

			columns.value = ['departmentName', 'total', 'inWork', 'success', 'fail']

			// --- Режим: по сотрудникам ---
		} else {
			console.log(user.UF_DEPARTMENT)
			console.log(selectedDepartment.value)

			if (
				selectedDepartment.value &&
				!(user.UF_DEPARTMENT || []).includes(+selectedDepartment.value)
			)
				continue

			if (!map[user.ID]) {
				map[user.ID] = {
					key: user.ID,
					userName: `${user.NAME} ${user.LAST_NAME}`,
					total: 0,
					inWork: 0,
					success: 0,
					fail: 0,
					id: user.ID,
				}
			}

			map[user.ID].total++
			if (deal.STAGE_SEMANTIC_ID === 'P') map[user.ID].inWork++
			if (deal.STAGE_SEMANTIC_ID === 'S') map[user.ID].success++
			if (deal.STAGE_SEMANTIC_ID === 'F') map[user.ID].fail++

			columns.value = ['userName', 'total', 'inWork', 'success', 'fail']
		}
	}

	// Оставляем только те строки, где есть хотя бы одна сделка
	rows.value = Object.values(map).filter(r => r.total > 0)

	isReportBuilt.value = true
	isLoading.value = false
}

onMounted(async () => {
	await sleep(1000)

	try {
		users.value = await loadUsers()
		departments.value = await loadDepartments()
		deals.value = await loadDeals()

		updateDepartmentItems()
	} catch (e: unknown) {
	} finally {
		isAppLoading.value = false
	}
})
</script>

<style scoped>
.app-start-box {
	width: 100%;
	height: 100dvh;
	display: grid;
	place-items: center;
}

.table-info {
	font-size: 13px;
	color: #555;
	font-style: italic;
}

.container {
	margin-top: 1rem;
	margin-bottom: 2rem;
}
.filters-box {
	display: flex;
	gap: 1rem;
	margin-bottom: 1rem;
}

.filter-group {
	margin-top: 1rem;
	margin-bottom: 1rem;
}

.filter-title {
	font-weight: 600;
	margin-bottom: 4px;
	margin-top: 1rem;
}

.filter-subtitle {
	font-size: 12px;
	color: #888;
	margin-bottom: -8px;
}

.filter-options {
	display: flex;
	gap: 1rem;
	flex-wrap: wrap;
	margin-left: -10px;
}

.filter-block {
	display: flex;
	flex-direction: column;
	gap: 8px;
}

.info-box {
	display: flex;
	align-items: center;
	gap: 8px;
	margin-bottom: 1rem;
	color: #555;
}

.empty-state {
	text-align: center;
	margin-top: 2rem;
	color: #777;
}

.text-disabled {
	cursor: not-allowed;
	user-select: none;
	pointer-events: none;
}

.select-item {
	display: flex;
	justify-content: space-between;
	padding: 5px 15px;
	cursor: pointer;
}

.row {
	transition: .2s;
}

.row:hover {
	background-color: aliceblue;
}
</style>
