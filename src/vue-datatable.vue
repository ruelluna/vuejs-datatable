<template>
	<table :class="tableClass">
		<thead>
			<tr>
				<datatable-header
					v-for="(head_column, i) in normalizedColumns"
					:key="i"
					:column="head_column"
					:settings="settings"
					:direction="getSortDirectionForColumn(head_column)"
					@change="setSortDirectionForColumn" />
			</tr>
		</thead>
		<tbody>
			<slot
				v-for="(row, i) in displayedRows"
				:row="row"
				:columns="normalizedColumns">
				<tr
					:key="i"
					:class="getRowClasses(row)">
					<datatable-cell
						v-for="(column, j) in normalizedColumns"
						:key="j"
						:column="column"
						:row="row" />
				</tr>
			</slot>
			<tr v-if="displayedRows.length == 0">
				<td :colspan="normalizedColumns.length">
					<slot name="no-results" />
				</td>
			</tr>
		</tbody>
		<tfoot v-if="$slots.footer || $scopedSlots.footer">
			<slot
				name="footer"
				:rows="displayedRows" />
		</tfoot>
	</table>
</template>

<script>
import Column from './classes/column.js';
import { ensurePromise } from './utils';

/**
 * @typedef {Object} DataFnParams
 * @description Parameters passed to the `data` function, to handle by custom logic.
 * @property {string | stringp[]}    filter:  The string(s) used to filter entries.
 * @property {string | null}         sortBy:  The name of the field we are sorting on.
 * @property {'asc' | 'desc' | null} sortDir: The direction of the sort.
 * @property {number}                perPage: The number of items per page.
 * @property {number}                page:    The current page index.
 * @tutorial ajax-data
 */

/**
 * The main component of the module, used to display a datatable.
 * 
 * @module datatable
 * 
 * @vue-prop {string} [name]                                     - The name of the datatable. It should be unique per page.
 * @vue-prop {ColumnDef[]} columns                               - List of columns definitions displayed by this datatable.
 * @vue-prop {Array.<*>|Function} data                           - The list of items to display, or a getter function.
 * @vue-prop {string} [filter]                                   - Value to match in rows for display filtering.
 * @vue-prop {(string | Array.<string> | Function)} [rowClasses] - Class(es) or getter function to get row classes.
 *
 * @vue-data {Column | null} sortBy          - Column used for data sorting.
 * @vue-data {'asc' | 'desc' | null} sortDir - Direction of the sort. A null value is equivalent to 'asc'.
 * @vue-data {number} totalRows              - Total number of rows contained by this data table.
 * @vue-data {number} page                   - Current page index.
 * @vue-data {number | null} perPage         - Maximum number of rows displayed per page.
 * @vue-data {Row[]} displayedRows           - Array of rows displayed by the table.
 * @vue-data {datatable-pager[]} pagers      - Array of pagers that are linked to this table.
 * 
 * @vue-computed {Array.<*>} rows             - Array of rows currently managed by the datatable.
 * @vue-computed {Settings} settings          - Reference to the {@link Settings} object linked to this datatable instance.
 * @vue-computed {Handler} handler            - Reference to the {@link Handler} object linked to this datatable instance.
 * @vue-computed {Column[]} normalizedColumns - Array of columns definitions casted as {@link Column} objects.
 * @vue-computed {string} tableClass          - Base CSS class to apply to the `&lt;table&gt;` element.
 */
export default {
	props: {
		name: {
			type:    String,
			default: 'default',
		},
		waitForPager: {
			type:    Boolean,
			default: false,
		},
		columns: {
			type:     Array,
			required: true,
		},
		data: {
			type:     null,
			required: true,
		},
		filter: {
			type:    [ String, Array ],
			default: null,
		},
		rowClasses: {
			type:    [ String, Array, Function ],
			default: null,
		},
	},
	data: () => ( {
		sortBy:        null,
		sortDir:       null,
		totalRows:     0,
		page:          1,
		perPage:       null,
		displayedRows: [],
		pagers:        [],
	} ),
	computed: {
		settings(){
			return this.$options.settings;
		},
		handler(){
			return this.$options.handler;
		},
		normalizedColumns(){
			return this.columns.map( column => new Column( column ) );
		},
		tableClass(){
			return this.settings.get( 'table.class' );
		},
	},
	created(){
		this.$datatables[this.name] = this;
		this.$root.$emit( 'table.ready', this.name );
		this.$watch( 'data', this.processRows, {deep: true, immediate: false} );
		this.$watch( 'columns', this.processRows, {deep: true, immediate: false} );

		// Defer to next tick, so a pager component created just after have the time to link itself with this table before start watching.
		this.$nextTick( () => {
			if ( this.waitForPager && this.pagers.length === 0 ){
				this.$on( 'table.pager-bound', () => this.initWatchCriterions() );
			} else {
				this.initWatchCriterions();
			}
		} );
	},
	methods: {
		/**
		 * Get the sort direction for a specific column.
		 * 
		 * @param {Column} columnDefinition - The column to check sorting direction for.
		 * @returns {'asc' | 'desc' | null} The sort direction for the specified column.
		 */
		getSortDirectionForColumn( columnDefinition ){
			if ( this.sortBy !== columnDefinition ){
				return null;
			}

			return this.sortDir;
		},
		/**
		 * Defines the sort direction for a specific column.
		 * 
		 * @param {'asc' | 'desc' | null} direction - The direction of the sort.
		 * @param {Column} column                   - The column to check sorting direction for.
		 * @returns {void} Nothing.
		 */
		setSortDirectionForColumn( direction, column ){
			this.sortBy = column;
			this.sortDir = direction;
		},
		/**
		 * Using data (or its return value if it is a function), filter, sort, paginate & display rows in the table.
		 * 
		 * @returns {Promise<void>} Nothing.
		 * @see DataFnParams Parameters provided to the `data` function.
		 * @tutorial ajax-data
		 */
		processRows(){
			if ( typeof this.data === 'function' ){
				const params = {
					filter:  this.filter,
					sortBy:  this.sortBy ? this.sortBy.field : null,
					sortDir: this.sortDir,
					perPage: this.perPage,
					page:    this.page,
				};

				return ensurePromise( this.data( params ) )
					.then( tableContent => ensurePromise( this.setTableContent( tableContent ) ) );
			}

			const outObj =  { source: this.data };
			return ensurePromise( this.handler.filterHandler( this.data, this.filter, this.normalizedColumns ) )
				.then( filteredData => ensurePromise( this.handler.sortHandler( outObj.filtered = filteredData, this.sortBy, this.sortDir ) ) )
				.then( sortedData => ensurePromise( this.handler.paginateHandler( outObj.sorted = sortedData, this.perPage, this.page ) ) )
				.then( pagedData => ensurePromise( this.handler.displayHandler( Object.assign( {paged: pagedData}, outObj ) ) ) )
				.then( tableContent => ensurePromise( this.setTableContent( tableContent ) ) );
		},
		setTableContent( { rows, totalRowCount } = { rows: undefined, totalRowCount: undefined } ){
			this.setRows( rows );
			this.setTotalRowCount( totalRowCount );
		},
		/**
		 * Set the displayed rows.
		 * 
		 * @param {Row[]} rows - The rows to display.
		 * @returns {void} Nothing.
		 */
		setRows( rows ){
			if ( typeof rows !== 'undefined' && rows !== null ){
				this.displayedRows = rows;
			}
		},
		/**
		 * Set the displayed rows count.
		 * 
		 * @param {number} value - The number of displayed rows.
		 * @returns {void} Nothing.
		 */
		setTotalRowCount( value ){
			if ( typeof value !== 'undefined' && value !== null ){
				this.totalRows = value;
			}
		},
		/**
		 * Get the classes to add on the row
		 * 
		 * @param {Row} row - The row to get classes for.
		 * @returns {string} The classes string to set on the row.
		 */
		getRowClasses( row ){
			let rowClasses = this.rowClasses;

			if ( rowClasses === null ){
				rowClasses = this.settings.get( 'table.row.class' );
			}

			if ( typeof rowClasses === 'function' ){
				return rowClasses( row );
			}

			return rowClasses;
		},
		/**
		 * Starts the watching of following properties: `filter`, `perPage`, `page`, `sortBy`, `sortDir`.
		 * When a change is detected, the component runs {@link datatable#processRows}.
		 * Because the watch is immediate, {@link datatable#processRows} is run immediately when this method is called.
		 * 
		 * @see datatable#processRows
		 * @see https://vuejs.org/v2/api/#vm-watch
		 * @returns {void} Nothing.
		 */
		initWatchCriterions(){
			for ( const prop of [ 'filter', 'perPage', 'page', 'sortBy', 'sortDir' ] ){
				this.$watch( prop, this.processRows, { immediate: false, deep: true } );
			}
			this.processRows();
		},
	},
	handler:  null,
	settings: null,
};
</script>
