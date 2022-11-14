<template>
	<div class="base-grid-container">
		<div class="base-grid-header">
			<span v-if="showTitle" class="base-grid-header-title">{{
				title
			}}</span>
			<div v-if="showSearch" class="base-grid-header-search">
				<input
					v-model="searchInput"
					type="text"
					:placeholder="searchPlaceholder"
					@keypress.enter="onSearch"
				/>
				<img
					src="@/assets/svg/Search.svg"
					alt="search"
					@click="onSearch"
				/>
			</div>
		</div>
		<div class="base-grid-table-wrap">
			<table class="base-grid-table">
				<thead ref="head"></thead>
				<tbody ref="body">
					<tr class="empty-message" v-if="isShowEmptyMessage">
						<td colspan="100%">
							<div class="empty-record-title">
								검색 결과가 없습니다
							</div>
							<div class="empty-record-sub-title">
								검색어를 확인하고 다시 검색해 주세요
							</div>
						</td>
					</tr>
				</tbody>
			</table>
		</div>
	</div>
</template>

<script>
const MODE = {
	CELL: 'cell',
	INPUT: 'input',
}

export default {
	name: 'JwGrid',
	props: {
		mode: {
			type: String,
			default: MODE.CELL,
		},
		title: {
			type: String,
			default: '',
		},
		headerStyle: {
			type: Object,
			default: () => {
				return {
					borderRight: '1px solid #EEEEEE',
					borderBottom: '1px solid #eee',
					padding: '10px',
					minWidth: '120px',
					textAlign: 'center',
					background: '#F5F5F5',
				}
			},
		},
		bodyStyle: {
			type: Object,
			default: () => {
				return {
					borderRight: '1px solid #EEEEEE',
					borderBottom: '1px solid #eee',
					padding: '10px',
					minWidth: '120px',
					textAlign: 'center',
					background: 'white',
				}
			},
		},
		suffix: {
			type: String,
			default: '원',
		},
		inputPlaceholder: {
			type: String,
			default: '',
		},
		searchPlaceholder: {
			type: String,
			default: '',
		},
		fixedColumn: {
			type: Number,
			default: 0,
		},
		titleColumnSize: {
			type: Number,
			default: 1,
		},
		showTitle: {
			type: Boolean,
			default: true,
		},
		showSearch: {
			type: Boolean,
			default: true,
		},
		toggleTree: {
			type: Boolean,
			default: false,
		},
		readonly: {
			type: Boolean,
			default: false,
		},
		titleTreeDesign: {
			type: Boolean,
			default: false,
		},
		notReadonlyColumnSize: {
			type: Number,
			default: 1,
		},
		summaryKey: {
			type: String,
			default: '',
		},
		subSummaryKey: {
			type: String,
			default: '',
		},
		summaryTitleIndex: {
			type: Number,
			default: 0,
		},
		summaryStyle: {
			type: Object,
			default: () => {
				return {
					borderRight: '1px solid #EEEEEE',
					borderBottom: '1px solid #eee',
					padding: '10px',
					background: '#F4F9FC',
					fontWeight: 'bold',
					color: '#0075C5',
				}
			},
		},
		summaryTitleStyle: {
			type: Object,
			default: () => {},
		},
		summaryTitle: {
			type: String,
			default: '소계',
		},
		summaryMaxDepth: {
			type: Number,
			default: 0,
		},
		totalSummary: {
			type: Boolean,
			default: false,
		},
		totalSummaryTitle: {
			type: String,
			default: '총계',
		},
		rowSummary: {
			type: Boolean,
			default: false,
		},
		rowSummaryTitle: {
			type: String,
			default: '계',
		},
	},
	data() {
		return {
			indexField: {},
			recid: 0,
			tree: {},
			isShowEmptyMessage: false,
			searchInput: '',
			columns: [],
			flatColumns: [],
			records: [],
			options: {},
			fixedColumnSize: new Map(),
			summaryRowList: [],
			fixedColumnCopy: 0,
			titleColumnSizeCopy: 1,
			summaryTitleIndexCopy: 0,
			notReadonlyColumnSizeCopy: 1,
			maxDepth: 1,
		}
	},
	watch: {
		/**
		 * readonly = true인 경우 grid 스타일 적용
		 * readonly = false인 경우 그리드 redraw
		 *
		 * @param {boolean} value
		 */
		readonly(value) {
			if (value) {
				this.applyAllReadonlyStyle()
			} else {
				this.create(this.columns)
				this.setGridData(this.records, this.options)
			}
		},
	},
	methods: {
		/**
		 * grid 데이터 가져오기
		 *
		 * @returns grid 생성 시 넣었던 원본 데이터
		 */
		getRecords() {
			return this.removeRecordAddData(this.records)
		},
		/**
		 * 현재 그려진 grid의 데이터에서 원본 데이터를 제외한 데이터 삭제
		 *
		 * @param {Object[]} records grid 데이터
		 * @returns {Object[]} result 변환 데이터
		 */
		removeRecordAddData(records) {
			if (!records || records.length <= 0) return []

			let result = []
			let index = 0
			records.forEach(record => {
				if (record.isSummary) return

				result.push(record)
				delete result[index].recid
				delete result[index].rowSummary

				if (record.children && record.children.length > 0)
					result[index].children = this.removeRecordAddData(
						record.children
					)
				index++
			})
			return result
		},
		/**
		 * grid 헤더 생성
		 *
		 * @param {Object[]} columns grid 헤더 컬럼 목록
		 */
		create(columns) {
			this.columns = this.$root.deepCopy(columns)
			this.flatColumns = this.convertFlatColumns(columns)
			this.clearColumns()

			this.createGridHeader(columns)
			if (this.rowSummary) this.addRowSummaryHeader()

			this.setTablePosition()
			this.setIndexField()
		},
		/**
		 * 헤더 컬럼 목록 flat하게 배열로 반환
		 *
		 * @param {Object[]} columns grid 헤더 컬럼 목록
		 * @returns {string[]} flatColumns
		 */
		convertFlatColumns(columns) {
			let flatColumns = []
			columns.forEach(column => {
				if (column.field) flatColumns.push(column.field)
				if (column.children && column.children.length > 0)
					flatColumns = [
						...flatColumns,
						...this.convertFlatColumns(column.children),
					]
			})
			return flatColumns
		},
		/**
		 * 헤더 컬럼 모두 제거
		 */
		clearColumns() {
			this.removeNodeList('.base-grid-table thead tr')
		},
		/**
		 * grid 헤더 element 생성 재귀함수
		 *
		 * @param {Object[]} columns grid 헤더 컬럼 목록
		 */
		createGridHeader(columns) {
			let children = []
			let index = 0
			const tr = document.createElement('tr')
			columns.forEach(column => {
				const th = document.createElement('th')
				tr.appendChild(th)

				th.innerHTML = column.text ? column.text : ''
				if (column.field)
					th.setAttribute('data-grid-field', column.field)

				this.applyHeaderDefaultStyle(th)
				let dummyColspan = this.applyCellStyle(th, column, index)
				index += column.colSpan ? column.colSpan : 1
				if (dummyColspan > 0) {
					const dummyTh = document.createElement('th')
					dummyTh.innerHTML = th.innerHTML
					dummyTh.colSpan = dummyColspan
					dummyTh.setAttribute(
						'data-origin-col-span',
						dummyTh.colSpan
					)
					this.applyHeaderDefaultStyle(dummyTh)
					tr.appendChild(dummyTh)
				}

				if (column.children && column.children.length > 0)
					children = children.concat(column.children)
			})
			this.$refs.head.appendChild(tr)

			if (!this.isEmptyObjectAll(children))
				this.createGridHeader(children)
		},
		/**
		 * 행 합계 헤더 생성
		 */
		addRowSummaryHeader() {
			const tableEl = document.querySelector('.base-grid-table')
			const rowLength = tableEl.rows.length
			const header = tableEl.rows[0]

			let size = 0
			let summaryIndex = 0
			this.columns.forEach((column, index) => {
				size += column.colSpan ? column.colSpan : 1
				if (size === this.titleColumnSize) summaryIndex = index + 1
			})
			const cell = header.insertCell(summaryIndex)

			cell.innerHTML = this.rowSummaryTitle
			cell.rowSpan = rowLength
			cell.setAttribute('data-grid-field', 'rowSummary')
			this.applyHeaderDefaultStyle(cell)
		},
		/**
		 * grid 헤더 목록 필드값을 넣은 indexField 배열 생성
		 */
		setIndexField() {
			const columns = document.querySelectorAll('th[data-grid-field]')
			columns.forEach(column => {
				this.indexField[column.dataset.gridField] =
					JSON.parse(column.dataset.position).left + 1
			})
		},
		/**
		 * grid 생성
		 * grid 생성에 관한 모든 함수들 호출
		 *
		 * @param {Object[]} records grid 바디에 들어갈 데이터 목록
		 * @param {Object} options 열(필드)별 옵션
		 */
		setGridData(records, options) {
			this.records = this.removeRecordAddData(records)
			this.options = options
			this.fixedColumnCopy = this.fixedColumn
			this.titleColumnSizeCopy = this.titleColumnSize
			this.summaryTitleIndexCopy = this.summaryTitleIndex
			this.notReadonlyColumnSizeCopy = this.notReadonlyColumnSize

			this.addSummaryRecords(this.records)
			this.clearGrid()

			if (!this.records || this.records.length <= 0) {
				return this.showEmptyMessage()
			} else {
				this.hideEmptyMessage()
			}

			this.createGridBody(this.records)

			if (this.titleTreeDesign) this.applyTreeDesign()
			this.setTablePosition()
			this.autoMergeRowSpan()

			if (this.readonly) this.applyAllReadonlyStyle()
			if (this.mode === MODE.INPUT) {
				const nodeList = [
					...document.querySelectorAll('.base-grid-table td'),
				].filter(node => node.dataset.gridField !== 'rowSummary')
				nodeList.forEach(node => {
					const inputEl = node.children[0].children[0]
					if (
						JSON.parse(node.dataset.position).left <
							this.titleColumnSizeCopy &&
						node.children.length > 0 &&
						inputEl
					) {
						this.replaceTag(
							node,
							inputEl.tagName.toLowerCase(),
							inputEl.value
						)
					}
				})
			}

			this.initEvent()
			if (this.fixedColumnSize.size <= 0) {
				setTimeout(() => {
					this.applyfixedColumn()
				}, 400)
			} else {
				this.applyfixedColumn()
			}
		},
		/**
		 * grid 소계 데이터 추가 분기 함수
		 *
		 * @param {Object[]} records grid 데이터
		 */
		addSummaryRecords(records) {
			if (this.summaryMaxDepth > 0 || this.totalSummary)
				this.addTreeSummaryRecords(records)

			if (this.rowSummary) this.addRowSummaryRecords(records)
			else if (this.summaryKey) this.addSummaryRecordsByKey(records)

			if (this.subSummaryKey) this.addSubSummaryRecordsByKey(records)
		},
		/**
		 * 트리 구조 소계 데이터 추가
		 *
		 * @param {Object[]} records grid 데이터
		 * @param {number} depth 트리 깊이
		 * @returns {Object} totalSummaryMap 총계 map
		 */
		addTreeSummaryRecords(records, depth = 0) {
			let totalSummaryMap = new Map()
			let prevValue = {}
			let summaryList = []
			records.forEach((record, rowIndex) => {
				if (record.isSummary) return

				let summaryMap = new Map()
				let childrenSummaryMap = null
				Object.entries(record).forEach(([key, value], cellIndex) => {
					if (key === 'children' && value.length > 0) {
						childrenSummaryMap = this.addTreeSummaryRecords(
							value,
							depth + 1
						)
						for (const [
							childKey,
							childValue,
						] of childrenSummaryMap.entries()) {
							if (
								(this.indexField[childKey] &&
									typeof childValue === 'number') ||
								childKey === 'rowSummary'
							) {
								summaryMap.set(
									childKey,
									summaryMap.has(childKey)
										? summaryMap.get(childKey) + childValue
										: childValue
								)
								totalSummaryMap.set(
									childKey,
									totalSummaryMap.has(childKey)
										? totalSummaryMap.get(childKey) +
												childValue
										: childValue
								)
							}
						}
						return
					}

					if (
						(this.indexField[key] && typeof value === 'number') ||
						key === 'rowSummary'
					) {
						if (
							this.options &&
							!this.isEmptyObject(this.options) &&
							this.options[key] &&
							this.options[key].mergeRowSpan
						) {
							if (
								!prevValue[key] ||
								(rowIndex > 0 &&
									records[rowIndex - 1].isSummary)
							) {
								prevValue[key] = value
							}
						}

						summaryMap.set(
							key,
							summaryMap.has(key)
								? summaryMap.get(key) + value
								: value
						)
						totalSummaryMap.set(
							key,
							totalSummaryMap.has(key)
								? totalSummaryMap.get(key) + value
								: value
						)
					} else {
						if (!summaryMap.has(key)) {
							let val = 0
							if (cellIndex === 0) val = value
							summaryMap.set(key, val)
						}
						if (!totalSummaryMap.has(key)) {
							totalSummaryMap.set(key, 0)
						}
					}
				})

				let summary = {
					isSummary: true,
				}
				for (const [key, value] of summaryMap.entries()) {
					summary[key] = value
				}
				summaryList.push({ index: rowIndex, summary })
			})

			if (depth < this.summaryMaxDepth) {
				summaryList.forEach((obj, index) => {
					records.splice(obj.index + index, 0, obj.summary)
				})
			}

			let totalSummaryObj = {
				isSummary: true,
			}
			for (const [key, value] of totalSummaryMap.entries()) {
				totalSummaryObj[key] = value
			}
			if (this.totalSummary && depth === 0)
				records.splice(0, 0, totalSummaryObj)
			return totalSummaryMap
		},
		/**
		 * 행 소계 데이터 추가
		 *
		 * @param {Object[]} records grid 데이터
		 */
		addRowSummaryRecords(records) {
			records.forEach(record => {
				let rowSummary = 0
				Object.entries(record).forEach(([key, value]) => {
					if (key === 'children' && value.length > 0)
						return this.addRowSummaryRecords(value)
					if (!this.indexField[key]) return

					if (typeof value === 'number') rowSummary += value
				})
				record.rowSummary = rowSummary
			})
		},
		/**
		 * key 기준 소계 추가
		 *
		 * @param {Object[]} records grid 데이터
		 */
		addSummaryRecordsByKey(records) {
			let prevSummaryValue = ''
			let summaryMap = new Map()
			records.forEach((record, index) => {
				Object.entries(record).forEach(([key, value]) => {
					if (record.isSummary || !value) return

					if (key === this.summaryKey) {
						if (prevSummaryValue === value) {
							const obj = summaryMap.get(value)
							obj.nodeList.push(record)
							summaryMap.set(value, obj)
						} else {
							summaryMap.set(value, {
								index,
								nodeList: [record],
							})
						}
						prevSummaryValue = value
					}
				})
			})

			let index = 0
			for (const obj of summaryMap.values()) {
				let summary = {
					isSummary: true,
				}
				const prevValue = {}
				obj.nodeList.forEach(node => {
					Object.entries(node).forEach(([key, value]) => {
						if (key === 'children' && value.length > 0)
							return this.addSummaryRecordsByKey(value)

						if (this.indexField[key] && typeof value === 'number') {
							if (
								this.options &&
								!this.isEmptyObject(this.options) &&
								this.options[key] &&
								this.options[key].mergeRowSpan
							) {
								if (!prevValue[key]) prevValue[key] = value
								else if (prevValue[key] === value) return
							}

							!summary[key]
								? (summary[key] = value)
								: (summary[key] = summary[key] + value)
						} else {
							if (!summary[key]) summary[key] = value
						}
					})
				})
				records.splice(obj.index + index++, 0, summary)
			}
		},
		/**
		 * key 기준 소계 추가 (디자인 변경, 같은 depth에 중복데이터 있는 경우)
		 *
		 * @param {Object[]} records grid 데이터
		 */
		addSubSummaryRecordsByKey(records) {},
		/**
		 * grid 바디 비우기
		 */
		clearGrid() {
			this.recid = 0
			this.tree = {}
			this.fixedColumnSize = new Map()
			this.showEmptyMessage()
			this.removeNodeList('.base-grid-table tbody tr:not(.empty-message)')
		},
		/**
		 * empty 화면 보이기
		 */
		showEmptyMessage() {
			this.isShowEmptyMessage = true
			document.querySelector('.base-grid-table').style.height = '100%'
		},
		/**
		 * empty 화면 숨기기
		 */
		hideEmptyMessage() {
			this.isShowEmptyMessage = false
			document.querySelector('.base-grid-table').style.height = 'auto'
		},
		/**
		 * 선택한 노드 목록 제거
		 *
		 * @param {string} selector element 셀렉터
		 */
		removeNodeList(selector) {
			const nodeList = document.querySelectorAll(selector)
			nodeList.forEach(node => {
				node.remove()
			})
		},
		/**
		 * grid 바디 element 생성 재귀함수
		 *
		 * @param {Object[]} records grid 데이터
		 * @param {number} depth 트리 깊이
		 */
		createGridBody(records, depth = 0) {
			const childNode = []
			records.forEach((record, rowIndex) => {
				let columns = []
				const tr = document.createElement('tr')
				record.recid = ++this.recid
				tr.setAttribute('recid', record.recid)
				tr.setAttribute('data-depth', depth)

				for (const [key, value] of Object.entries(record)) {
					if (
						key === 'children' ||
						key === 'recid' ||
						key === 'isSummary'
					)
						continue

					let isSummary = key === 'rowSummary'
					const isHidden =
						this.flatColumns.indexOf(key) === -1 && !isSummary

					let val = value
					if (!val) {
						if (isSummary || record.isSummary)
							val = `0${this.suffix}`
						else if (this.mode === MODE.CELL) val = '-'
					}
					if (typeof value === 'number' && !isHidden) {
						val = this.convertNumberToCurrency(value)
						if (
							this.mode === MODE.CELL ||
							isSummary ||
							record.isSummary
						)
							val += this.suffix
					}
					columns.push({
						key,
						td: this.setColumnValue(
							document.createElement('td'),
							key,
							val,
							isHidden,
							record.isSummary
						),
					})
				}
				this.sortColumns(columns)

				let rowSummaryIndex = null
				columns.forEach(({ key, td }, cellIndex) => {
					if (key === 'rowSummary') rowSummaryIndex = cellIndex

					let isSummary = key === 'rowSummary' || record.isSummary
					if (
						this.totalSummary &&
						isSummary &&
						rowIndex === 0 &&
						cellIndex === 0
					) {
						td.children[0].innerHTML = this.totalSummaryTitle
						td.colSpan =
							this.summaryTitleIndex > 0
								? this.summaryTitleIndexCopy
								: 1
						td.setAttribute('data-origin-col-span', td.colSpan)
						for (let i = 1; i < this.summaryTitleIndexCopy; i++) {
							columns[i].td.style.display = 'none'
						}
					}

					if (isSummary && cellIndex >= this.summaryTitleIndexCopy) {
						this.applySummaryDefaultStyle(td)
						td.setAttribute('data-merge-row-span', false)
						if (
							this.summaryTitleIndex > 0 &&
							cellIndex === this.summaryTitleIndexCopy
						) {
							td.children[0].innerHTML = this.summaryTitle
							td.colSpan =
								this.titleColumnSizeCopy -
								this.summaryTitleIndexCopy
							td.setAttribute('data-origin-col-span', td.colSpan)
							for (
								let i = this.summaryTitleIndexCopy + 1;
								i < this.titleColumnSizeCopy;
								i++
							) {
								columns[i].td.style.display = 'none'
							}
						}
					} else {
						this.applyBodyDefaultStyle(td)
						td.setAttribute('data-merge-row-span', true)
					}

					if (
						this.options &&
						!this.isEmptyObject(this.options) &&
						this.options[key]
					) {
						this.applyCellStyle(td, this.options[key])
					}
				})

				if (rowSummaryIndex) {
					columns.splice(
						this.titleColumnSizeCopy,
						0,
						columns[rowSummaryIndex]
					)
					columns.splice(rowSummaryIndex + 1, 1)
				}

				columns.forEach(({ td }) => {
					tr.appendChild(td)
				})
				this.$refs.body.appendChild(tr)
				childNode.push(tr)

				this.tree[tr.getAttribute('recid')] =
					record.children && record.children.length > 0
						? this.createGridBody(record.children, depth + 1)
						: []
			})
			return childNode
		},
		/**
		 * 데이터 표현 element 생성
		 *
		 * @param {HTMLElement} el 부모 element
		 * @param {string} key 필드
		 * @param {string} value 값
		 * @param {boolean} isHidden 숨김 여부
		 * @param {boolean} isSummary 소계 여부
		 * @returns {HTMLElement} el 작업 후 element 반환
		 */
		setColumnValue(el, key, value, isHidden, isSummary) {
			el.setAttribute('data-grid-field', key)
			if (
				this.mode === MODE.INPUT &&
				key !== 'rowSummary' &&
				!isSummary
			) {
				el.innerHTML = `
                    <div title="${value}">
                        <input placeholder="${this.inputPlaceholder}" value="${value}" maxlength="13"
                        style="border: 1px solid #DDDDDD; border-radius: 5px; width: 110px; height: 30px;">
                    </div>
                `
			} else {
				el.innerHTML = `<div title="${value}">${value}</div>`
			}
			if (isHidden) el.style.display = 'none'
			return el
		},
		/**
		 * grid 열 정렬
		 *
		 * @param {Object[]} columns 열 목록
		 */
		sortColumns(columns) {
			columns.sort((a, b) => {
				if (!this.indexField[a.key]) return 1
				if (!this.indexField[b.key]) return -1
				if (a.key === b.key) return 0
				return this.indexField[a.key] - this.indexField[b.key]
			})
		},
		/**
		 * cell에 옵션 적용
		 *
		 * @param {HTMLElement} el 스타일 적용할 cell
		 * @param {Object} option 옵션
		 * @param {number} index 순서
		 * @returns dummyColSpan colSpan 적용 시 fixedColumn 사이즈를 넘어가는 col 사이즈
		 */
		applyCellStyle(el, option, index) {
			el.style.width = option.size ? option.size : '120px'
			if (option.style) this.applyStyle(el, option.style)

			el.setAttribute('rowSpan', option.rowSpan ? option.rowSpan : 1)

			let dummyColSpan = 0
			let colSpan = el.colSpan
			if (option.colSpan) {
				colSpan = option.colSpan

				if (index < this.fixedColumnCopy) {
					const idx = index + option.colSpan
					if (idx > this.fixedColumnCopy) {
						const overSize = idx - this.fixedColumnCopy
						colSpan = option.colSpan - overSize
						dummyColSpan = overSize
					}
				}
			}
			el.colSpan = colSpan
			el.setAttribute('data-origin-col-span', el.colSpan)
			return dummyColSpan
		},
		/**
		 * 헤더 기본 스타일 적용
		 *
		 * @param {HTMLElement} el 스타일 적용할 cell
		 */
		applyHeaderDefaultStyle(el) {
			this.applyStyle(el, this.headerStyle)
		},
		/**
		 * 바디 기본 스타일 적용
		 *
		 * @param {HTMLElement} el 스타일 적용할 cell
		 */
		applyBodyDefaultStyle(el) {
			this.applyStyle(el, this.bodyStyle)
		},
		/**
		 * 소계 기본 스타일 적용
		 *
		 * @param {HTMLElement} el 스타일 적용할 cell
		 */
		applySummaryDefaultStyle(el) {
			this.applyStyle(el, this.summaryStyle)
		},
		/**
		 * cell에 스타일 적용
		 *
		 * @param {HTMLElement} el 스타일 적용할 cell
		 * @param {Object} style 적용할 스타일
		 */
		applyStyle(el, style) {
			Object.entries(style).forEach(([key, value]) => {
				if (el.children[0] && key === 'color')
					el.children[0].style[key] = value
				el.style[key] = value
			})
		},
		/**
		 * 모든 grid 영역 readonly 스타일 적용
		 */
		applyAllReadonlyStyle() {
			const columns = [
				...document.querySelectorAll(
					'.base-grid-table th, .base-grid-table td'
				),
			].filter(
				column =>
					JSON.parse(column.dataset.position).left >=
					this.notReadonlyColumnSizeCopy
			)

			columns.forEach(column => {
				column.style.color = '#CACBCB'
				column.style.background = '#F9F9F9'
				if (column.children[0]) {
					column.children[0].innerHTML =
						'<div style="width: 100%;"></div>'
				}
			})
		},
		/**
		 * grid 바디 타이틀 부분 트리 디자인 적용
		 */
		applyTreeDesign() {
			const nodeList = [
				...document.querySelectorAll(
					'.base-grid-table tbody tr:not(.empty-message)'
				),
			]
			this.maxDepth = this.getMaxDepth(nodeList) + 1
			const thList = [
				...document.querySelectorAll(
					'.base-grid-table thead tr th:first-of-type'
				),
			].reverse()

			let firstTh = null
			thList.forEach(th => {
				if (JSON.parse(th.dataset.position).left === 0) {
					if (firstTh == null) {
						firstTh = th
					} else if (this.maxDepth > firstTh.colSpan) {
						th.colSpan =
							th.colSpan + (this.maxDepth - firstTh.colSpan)
						th.setAttribute('data-origin-col-span', th.colSpan)
					}
				}
			})
			if (this.maxDepth && firstTh && this.maxDepth > firstTh.colSpan) {
				firstTh.colSpan = this.maxDepth
				firstTh.setAttribute('data-origin-col-span', firstTh.colSpan)
				this.fixedColumnCopy += this.maxDepth - 1
				this.titleColumnSizeCopy += this.maxDepth - 1
				this.summaryTitleIndexCopy += this.maxDepth - 1
				this.notReadonlyColumnSizeCopy += this.maxDepth - 1
			}

			let prevNode = null
			let prevDepth = 0
			nodeList.forEach((node, index) => {
				if (!node.children[0]) return

				const td = node.children[0]
				const fieldName = td.dataset.gridField
				const depth = parseInt(node.dataset.depth)
				td.colSpan = this.maxDepth - depth
				td.setAttribute('data-origin-col-span', td.colSpan)

				if (depth != 0) {
					for (let i = 0; i < depth; i++) {
						let style = this.$root.deepCopy(this.bodyStyle)
						if (
							this.options &&
							!this.isEmptyObject(this.options) &&
							this.options[fieldName]
						) {
							Object.assign(style, this.options[fieldName].style)
						}
						Object.assign(style, {
							borderBottom: 'none',
							minWidth: '50px',
						})

						const nextNode = nodeList[index + 1]
						let nextDepth = 0
						if (nextNode)
							nextDepth = parseInt(nextNode.dataset.depth)

						if (td.children[0].innerHTML) {
							if (prevDepth !== depth) {
								prevNode.style.borderBottom = 'none'
								td.style.borderTop = this.bodyStyle.borderBottom
							}
							if (nextNode && nextDepth > depth)
								td.style.borderBottom = 'none'
						}
						this.createCell(node, i, '', '', style)
					}
				}
				prevNode = td
				prevDepth = depth
			})
		},
		/**
		 * grid 트리 최대 깊이 값 반환
		 *
		 * @param {HTMLElement[]} list grid row element 목록
		 * @returns {number} maxDepth 최대 깊이
		 */
		getMaxDepth(list) {
			const depthList = list.map(node => parseInt(node.dataset.depth))
			return Math.max(...depthList)
		},
		/**
		 * cell 생성하기
		 *
		 * @param {HTMLElement} row 새로 생성한 소계 행
		 * @param {number} index 열 생성할 위치
		 * @param {string} value 표시할 데이터
		 * @param {string} fieldName 필드명
		 * @param {Object} style 적용할 스타일
		 * @param {number} colSpan 열 길이
		 * @returns {HTMLElement} cell 생성한 열 반환
		 */
		createCell(row, index, value, fieldName, style, colSpan) {
			const cell = row.insertCell(index)
			const div = document.createElement('div')
			div.innerHTML = value
			cell.setAttribute('data-grid-field', fieldName)
			cell.appendChild(div)

			this.applyStyle(
				cell,
				style && !this.isEmptyObject(style) ? style : this.summaryStyle
			)
			if (colSpan) {
				cell.colSpan = colSpan
				cell.setAttribute('data-origin-col-span', cell.colSpan)
			}
			return cell
		},
		/**
		 * grid 각 cell마다 커스텀 필드에 위치 지정
		 */
		setTablePosition() {
			const temp = []
			const nodeList = document.querySelectorAll('.base-grid-table tr')
			nodeList.forEach((row, y) => {
				row.children.forEach((cell, x) => {
					if (
						getComputedStyle(cell).display === 'none' &&
						cell.dataset.gridField !== this.summaryKey
					) {
						let left = 0
						if (x !== 0) {
							const prevCell = cell.previousElementSibling
							left =
								JSON.parse(prevCell.dataset.position).left + 1
						}

						cell.setAttribute(
							'data-position',
							JSON.stringify({
								top: y,
								left,
							})
						)
						return
					}

					let tx = 0
					let ty = 0
					for (; temp[y] && temp[y][x]; ++x);
					for (tx = x; tx < x + cell.colSpan; ++tx) {
						for (ty = y; ty < y + cell.rowSpan; ++ty) {
							if (!temp[ty]) {
								temp[ty] = []
							}
							temp[ty][tx] = true
						}
					}

					cell.setAttribute(
						'data-position',
						JSON.stringify({ top: y, left: x })
					)
				})
			})
		},
		/**
		 * option에 mergeRowSpan이 있는 열은 값이 같은 경우 자동 rowSpan 적용
		 */
		autoMergeRowSpan() {
			if (!this.options || this.isEmptyObject(this.options)) return

			const keys = Object.entries(this.options)
				.filter(([_, value]) => value.mergeRowSpan)
				.map(([key]) => key)

			keys.forEach(key => {
				const nodeList = document.querySelectorAll(
					`.base-grid-container td[data-grid-field="${key}"]`
				)

				let rowSpan = 1
				let spanNode = null
				let prevRecid = null
				let prevDepth = null
				let prevValue = null
				nodeList.forEach(node => {
					const tr = node.closest('tr')
					const recid = parseInt(tr.getAttribute('recid'))
					const depth = tr.dataset.depth
					const value = node.children[0].innerHTML
					const prevRow = tr.previousElementSibling

					if (
						recid === prevRecid + 1 &&
						depth === prevDepth &&
						prevValue === value &&
						node.dataset.mergeRowSpan == 'true' &&
						prevRow &&
						prevRow.children[node.cellIndex].dataset.mergeRowSpan ==
							'true'
					) {
						node.style.display = 'none'
						node.setAttribute('data-success-merge', true)
						rowSpan++
					} else {
						if (spanNode) spanNode.rowSpan = rowSpan
						if (
							rowSpan > 1 &&
							spanNode.cellIndex < this.titleColumnSizeCopy &&
							this.titleTreeDesign &&
							spanNode.dataset?.mergeRowSpan &&
							spanNode.parentElement.dataset.depth < depth
						)
							spanNode.style.borderBottom = 'none' // 트리 디자인 적용

						spanNode = node
						rowSpan = 1
					}

					prevRecid = recid
					prevDepth = depth
					prevValue = value
				})
				if (spanNode) spanNode.rowSpan = rowSpan
			})
		},
		/**
		 * 지정한 크기만큼의 열은 스크롤해도 위치 고정
		 */
		applyfixedColumn() {
			if (this.fixedColumn <= 0) return

			const tableRectLeft = document
				.querySelector('.base-grid-table')
				.getBoundingClientRect().left

			for (let i = 0; i < this.fixedColumnCopy; i++) {
				const columns = [
					...document.querySelectorAll(
						'.base-grid-table th, .base-grid-table td'
					),
				]
				columns
					.filter(
						column => JSON.parse(column.dataset.position).left === i
					)
					.forEach(column => {
						const fieldName = column.dataset.position
						if (!this.fixedColumnSize.has(fieldName)) {
							this.fixedColumnSize.set(
								fieldName,
								column.getBoundingClientRect().left -
									tableRectLeft
							)
						}
						column.style.left = `${this.fixedColumnSize.get(
							fieldName
						)}px`
						column.style.position = 'sticky'
					})
			}
		},
		/**
		 * 이벤트 적용 함수
		 */
		initEvent() {
			this.addCellClickEvent()
			this.addFirstCellClickEvent()
			if (this.mode === MODE.INPUT) {
				const inputEls = document.querySelectorAll(
					'.base-grid-container .base-grid-table-wrap input'
				)
				this.addInputEvent(inputEls)
			}
		},
		/**
		 * cell click 이벤트
		 */
		addCellClickEvent() {
			const cellList = [
				...document.querySelectorAll(
					'.base-grid-container tbody td[data-position]'
				),
			].filter(
				cell => JSON.parse(cell.dataset.position).left >= this.maxDepth
			)
			cellList.forEach(cell => {
				cell.addEventListener('click', e => {
					if (e.target.tagName !== 'INPUT') this.$emit('cellClick', e)
				})
			})
		},
		/**
		 * 첫번째 cell(타이틀) click 이벤트
		 */
		addFirstCellClickEvent() {
			const firstCellList = [
				...document.querySelectorAll(
					'.base-grid-container tbody td[data-position]'
				),
			].filter(
				cell => JSON.parse(cell.dataset.position).left < this.maxDepth
			)
			firstCellList.forEach(cell => {
				cell.addEventListener('click', e => {
					if (this.toggleTree) {
						this.toggleChildNode(
							e.target.parentNode.getAttribute('recid')
								? e.target.parentNode.getAttribute('recid')
								: e.target.parentNode.parentNode.getAttribute(
										'recid'
								  )
						)
					}
				})
			})
		},
		/**
		 * 입력 이벤트 적용 함수
		 *
		 * @param {HTMLElement[]} inputEls input element 목록
		 */
		addInputEvent(inputEls) {
			inputEls.forEach(inputEl => {
				this.addKeyInputEvent(inputEl)
				this.addKeyupEvent(inputEl)
				this.addMouseOverEvent(inputEl)
				this.addMouseLeaveEvent(inputEl)
				this.addFocusEvent(inputEl)
				this.addBlurEvent(inputEl)
			})
		},
		/**
		 * input 이벤트
		 *
		 * @param {HTMLElement} inputEl input element
		 */
		addKeyInputEvent(inputEl) {
			inputEl.addEventListener('input', e => {
				if (!e.target.value) return

				e.target.value = this.convertNumberToCurrency(
					e.target.value.replace(/[^0-9 \,]/, '')
				)
			})
		},
		/**
		 * keyup 이벤트
		 *
		 * @param {HTMLElement} inputEl input element
		 */
		addKeyupEvent(inputEl) {
			inputEl.addEventListener('keyup', e => {
				if (e.key === 'Enter') {
					inputEl.removeEventListener('blur', this.redrawEvent)
					this.redrawEvent(e)
				}
			})
		},
		/**
		 * mouseover 이벤트
		 *
		 * @param {HTMLElement} inputEl input element
		 */
		addMouseOverEvent(inputEl) {
			inputEl.addEventListener('mouseover', this.changeBorderFocus)
		},
		/**
		 * mouseleave 이벤트
		 *
		 * @param {HTMLElement} inputEl input element
		 */
		addMouseLeaveEvent(inputEl) {
			inputEl.addEventListener('mouseleave', this.changeBorderFocusOut)
		},
		/**
		 * focus 이벤트
		 *
		 * @param {HTMLElement} inputEl input element
		 */
		addFocusEvent(inputEl) {
			inputEl.addEventListener('focus', e => {
				inputEl.removeEventListener(
					'mouseleave',
					this.changeBorderFocusOut
				)
				this.changeBorderFocus(e)
			})
		},
		/**
		 * blur 이벤트
		 *
		 * @param {HTMLElement} inputEl input element
		 */
		addBlurEvent(inputEl) {
			inputEl.addEventListener('blur', this.redrawEvent)
		},
		/**
		 * border focus 스타일 적용
		 *
		 * @param {Event} e input event 객체
		 */
		changeBorderFocus(e) {
			e.target.style.border = '1px solid #333333'
		},
		/**
		 * border focusout 스타일 적용
		 *
		 * @param {Event} e input event 객체
		 */
		changeBorderFocusOut(e) {
			e.target.style.border = '1px solid #DDDDDD'
		},
		/**
		 * 값을 변경한 경우 grid 데이터(records)에 해당 값 변경 후
		 * grid 다시 생성
		 *
		 * @param {Event} e input event 객체
		 */
		redrawEvent(e) {
			const recid = e.target.closest('tr').getAttribute('recid')
			const field = e.target.closest('td').dataset.gridField
			const value = e.target.value
			this.changeRecordValue(recid, field, value)

			this.create(this.columns)
			this.setGridData(this.records, this.options)
		},
		/**
		 * grid 값 변경
		 *
		 * @param {number} recid grid 행의 recid
		 * @param {string} field 필드명
		 * @param {string} value 변경된 값
		 */
		changeRecordValue(recid, field, value) {
			const record = this.findRecordByRecid(this.records, recid)
			record[field] = value ? this.convertCurrencyToNumber(value) : ''
		},
		/**
		 * recid로 records 에서 원하는 record 가져옴
		 *
		 * @param {Object[]} records grid 데이터
		 * @param {number} recid grid 행의 recid
		 * @returns {Object} result grid record
		 */
		findRecordByRecid(records, recid) {
			let result = {}
			records.some(record => {
				if (record.recid == recid) {
					result = record
					return true
				} else if (record.children) {
					result = this.findRecordByRecid(record.children, recid)
					if (!this.isEmptyObject(result)) return true
				}
			})
			return result
		},
		/**
		 * 모든 트리 펼치기
		 */
		expandAll() {
			Object.entries(this.tree).forEach(([key]) => {
				this.toggleChildNode(key, 'table-row')
			})
		},
		/**
		 * 모든 트리 접기
		 */
		collapseAll() {
			Object.entries(this.tree).forEach(([key]) => {
				this.toggleChildNode(key, 'none')
			})
		},
		/**
		 * 원하는 트리의 자식노드 토글
		 *
		 * @param {number} recid 부모 트리의 recid
		 * @param {string} display display 스타일 값
		 */
		toggleChildNode(recid, display) {
			const children = this.tree[recid]
			if (!children) return

			children.forEach(el => {
				if (display) {
					el.style.display = display
				} else {
					el.style.display =
						el.style.display === 'none' ? 'table-row' : 'none'
				}
				this.toggleChildNode(el.getAttribute('recid'), el.style.display)
			})
		},
		/**
		 * 열 보이기
		 *
		 * @param {string[]} keys 보여줄 열 필드명 목록
		 */
		showColumns(keys) {
			keys.forEach(key => {
				if (!this.indexField[key]) return

				const columns = [
					...document.querySelectorAll(
						`.base-grid-container td[data-grid-field="${key}"],
                    .base-grid-container th[data-grid-field="${key}"]`
					),
				]
					.filter(
						column => getComputedStyle(column).display === 'none'
					)
					.filter(column => !column.dataset?.successMerge)
				columns.forEach(column => {
					column.style.display = 'table-cell'
					if (column.tagName === 'TH') {
						const row = column.parentElement
						const parentRowList = [
							...document.querySelectorAll(
								`.base-grid-table thead tr:nth-child(-n+${row.rowIndex})`
							),
						]

						if (parentRowList.length <= 0) return

						let targetSize = JSON.parse(
							column.dataset.position
						).left
						parentRowList.forEach(parentRow => {
							for (const th of parentRow.children) {
								const size = JSON.parse(
									th.dataset.position
								).left
								if (size >= targetSize) {
									const prevTh = th.previousElementSibling
									const prevSize =
										JSON.parse(prevTh.dataset.position)
											.left +
										parseInt(prevTh.dataset.originColSpan) -
										1
									if (prevSize >= targetSize) {
										getComputedStyle(prevTh).display ===
										'none'
											? (prevTh.style.display =
													'table-cell')
											: (prevTh.colSpan += 1)
										break
									}
								}
							}
						})
					}
				})
			})
		},
		/**
		 * 열 숨기기
		 *
		 * @param {string[]} keys 숨길 열 필드명 목록
		 */
		hideColumns(keys) {
			keys.forEach(key => {
				if (!this.indexField[key]) return

				const columns = [
					...document.querySelectorAll(
						`.base-grid-container td[data-grid-field="${key}"],
                    .base-grid-container th[data-grid-field="${key}"]`
					),
				].filter(column => getComputedStyle(column).display !== 'none')
				columns.forEach(column => {
					column.style.display = 'none'
					if (column.tagName === 'TH') {
						const row = column.parentElement
						const parentRowList = [
							...document.querySelectorAll(
								`.base-grid-table thead tr:nth-child(-n+${row.rowIndex})`
							),
						]

						if (parentRowList.length <= 0) return

						let targetSize = JSON.parse(
							column.dataset.position
						).left
						parentRowList.forEach(parentRow => {
							for (const th of parentRow.children) {
								if (getComputedStyle(th).display === 'none')
									continue

								let size = JSON.parse(th.dataset.position).left
								if (size >= targetSize) {
									th.colSpan === 1
										? (th.style.display = 'none')
										: (th.colSpan -= 1)
									break
								} else {
									size +=
										parseInt(th.dataset.originColSpan) - 1
									if (size >= targetSize) {
										th.colSpan === 1
											? (th.style.display = 'none')
											: (th.colSpan -= 1)
										break
									}
								}
							}
						})
					}
				})
			})
		},
		/**
		 * 하이라이트 검색 이벤트
		 */
		onSearch() {
			if (!this.isShowEmptyMessage) this.toHighlightNode()
		},
		/**
		 * 검색한 값에 해당하는 노드에 하이라이팅 적용
		 */
		toHighlightNode() {
			this.clearHighlight()

			const nodeList = [
				...document.querySelectorAll('.base-grid-table td'),
			].filter(node => getComputedStyle(node).display !== 'none')

			const findNodeList = nodeList
				.map(node => node.children[0])
				.filter(node => node)
				.filter(node => !node.children[0])
				.filter(node => node.innerHTML.includes(this.searchInput))
			if (findNodeList.length <= 0) return

			findNodeList.forEach(node => {
				const matchArr = [
					...node.innerHTML.matchAll(
						new RegExp(this.searchInput, 'gi')
					),
				]
				let highlightLength = 0
				matchArr.forEach(matchNode => {
					const startIndex = matchNode.index + highlightLength
					const endIndex = startIndex + this.searchInput.length
					const highlightEl = `
                        <span class="base-grid-table-highlight" style="background: rgba(255, 87, 70, 0.5);">${this.searchInput}</span>
                    `.trim()
					highlightLength += highlightEl.length - 1

					node.innerHTML =
						node.innerHTML.substring(0, startIndex) +
						highlightEl +
						node.innerHTML.substring(endIndex)
				})
			})
		},
		/**
		 * 하이라이팅 지우기
		 */
		clearHighlight() {
			const highlightNodeList = document.querySelectorAll(
				'.base-grid-table-highlight'
			)
			if (highlightNodeList.length <= 0) return

			highlightNodeList.forEach(highlightNode => {
				this.replaceTag(highlightNode, 'span', '')
			})
		},
		/**
		 * 태그를 원하는 값으로 교체
		 *
		 * @param {HTMLElement} el 태그 변경할 element
		 * @param {string} tag 없앨 태그
		 * @param {string} replaceValue 변경할 값
		 */
		replaceTag(el, tag, replaceValue) {
			const regex = new RegExp(`<(\/${tag}|${tag})([^>]*)>`, 'gi')
			el.outerHTML = el.outerHTML.replace(regex, replaceValue)
		},
		/**
		 * 숫자를 금액으로 변환
		 *
		 * @param {number} value 숫자
		 * @returns {string} currency 금액
		 */
		convertNumberToCurrency(value) {
			if (!value) return '0'
			if (value == 0) return value

			return this.convertCurrencyToNumber(value)
				.toString()
				.replace(/\B(?=(\d{3})+(?!\d))/g, ',')
		},
		/**
		 * 금액을 숫자로 변환
		 *
		 * @param {string} value 금액
		 * @returns {number} number 숫자
		 */
		convertCurrencyToNumber(value) {
			if (!value) return '0'
			if (value == 0) return value

			return parseInt(value.toString().replace(/,/g, ''))
		},
		/**
		 * 모든 객체가 비어있는지 확인
		 *
		 * @param {Object[]} arr 검증할 객체 목록
		 * @returns {boolean} isEmpty
		 */
		isEmptyObjectAll(arr) {
			return arr.every(obj => this.isEmptyObject(obj))
		},
		/**
		 * 해당 객체가 비어있는지 확인
		 *
		 * @param {Object} arr 검증할 객체
		 * @returns {boolean} isEmpty
		 */
		isEmptyObject(obj) {
			return Object.keys(obj).length === 0 && obj.constructor === Object
		},
	},
}
</script>

<style lang="scss" scoped>
.base-grid-container {
	display: flex;
	flex-direction: column;
	position: relative;
	width: 100%;
	white-space: nowrap;
	.base-grid-header {
		display: flex;
		justify-content: space-between;
		margin-bottom: 15px;
		.base-grid-header-title {
			font-size: 16px;
			font-weight: bold;
		}
		.base-grid-header-search {
			display: flex;
			input {
				width: 270px;
				height: 30px;
				border: 1px solid $INPUT_BORDER;
				border-radius: 4px 0 0 4px;
				font-size: 14px;
			}
			img {
				width: 30px;
				height: 30px;
				background: $COMMON_BACKGROUND;
				border: 1px solid $INPUT_BORDER;
				border-left: none;
				border-radius: 0 4px 4px 0;
				box-sizing: border-box;
				cursor: pointer;
			}
		}
	}
	.base-grid-table-wrap {
		overflow: auto;
		border-radius: 5px;
		border: 1px solid $COLOR_DDD;
		.base-grid-table {
			table-layout: fixed;
			font-size: 14px;
			border-collapse: separate; /* Don't collapse */
			border-spacing: 0;
			thead {
				position: sticky;
				top: 0;
				z-index: 2;
				background: $TABLE_HEADER;
				color: $COLOR_666;
				font-weight: bold;
			}
			tbody {
				background: $PRIMARY_WHITE;
				color: $COLOR_111;
				font-weight: normal;
				font-size: 0.875rem;
				.empty-record-title {
					font-size: 16px;
					font-weight: bold;
					color: $COLOR_666;
				}
				.empty-record-sub-title {
					font-size: 14px;
					color: $COLOR_999;
				}
			}
		}
	}
}
</style>
