<template>
    <table
        cellspacing="0"
        cellpadding="0"
        class="date-table"
        @click="handleClick"
        @mousemove="handleMouseMove">
        <tbody>
            <tr>
                <th v-if="showWeekNumber"></th>
                <th v-for="week in WEEKS">{{week}}</th>
            </tr>
            <tr
                class="date-table__row"
                v-for="row in rows">
                <td
                    v-for="cell in row"
                    :class="getCellClasses(cell)">
                    <div>
                        <span>
                            {{ cell.type==='today' ? '今' : cell.text }}
                        </span>
                    </div>
                </td>
            </tr>
        </tbody>
    </table>
</template>

<script>
    import { getFirstDayOfMonth, getDayCountOfMonth, getWeekNumber, getStartDateOfMonth, nextDate } from '../../utils/date';
    import { hasClass } from '../../utils/dom';

    const WEEKS = ['日', '一', '二', '三', '四', '五', '六'];
    const clearHours = function(time) {
        const cloneDate = new Date(time);

        cloneDate.setHours(0, 0, 0, 0);

        return cloneDate.getTime();
    };

    export default {
        name: 'date-table',

        props: {
            // 一周的第一天
            firstDayOfWeek: {
                default: 7,
                type: Number,
                validator: val => val >= 1 && val <= 7,
            },

            value: {},

            date: {},

            selectionMode: {
                type: String,
                default: 'day',
            },
            // 是否展示周
            showWeekNumber: {
                type: Boolean,
                default: false,
            },

            // 禁选日期
            disabledDate: {},

            selectedDate: {
                type: Array,
            },

            minDate: {},

            maxDate: {},

            // 日期区间
            rangeState: {
                default() {
                    return {
                        endDate: null,
                        selecting: false,
                        row: null,
                        column: null,
                    };
                },
            },
        },

        computed: {
            offsetDay() {
                const week = this.firstDayOfWeek;

                // 周日为界限，左右偏移的天数，3217654 例如周一就是 -1，目的是调整前两行日期的位置

                return week > 3 ? 7 - week : -week;
            },

            WEEKS() {
                const week = this.firstDayOfWeek;

                return WEEKS.concat(WEEKS).slice(week, week + 7);
            },

            year() {
                return this.date.getFullYear();
            },

            month() {
                return this.date.getMonth();
            },

            startDate() {
                return getStartDateOfMonth(this.year, this.month);
            },

            rows() {
                // TODO: refactory rows / getCellClasses
                const date = new Date(this.year, this.month, 1);
                let day = getFirstDayOfMonth(date); // day of first day
                const dateCountOfMonth = getDayCountOfMonth(date.getFullYear(), date.getMonth());
                const dateCountOfLastMonth = getDayCountOfMonth(
                                                date.getFullYear(),
                                                (date.getMonth() === 0 ? 11 : date.getMonth() - 1));

                day = (day === 0 ? 7 : day);

                const offset = this.offsetDay;
                const rows = this.tableRows;
                let count = 1;
                let firstDayPosition;

                const startDate = this.startDate;
                const disabledDate = this.disabledDate;
                const selectedDate = this.selectedDate || this.value;
                const now = clearHours(new Date());

                for (let i = 0; i < 6; i++) {
                    const row = rows[i];

                    if(this.showWeekNumber) {
                        if(!row[0]) {
                            row[0] = { type: 'week', text: getWeekNumber(nextDate(startDate, i * 7 + 1)) };
                        }
                    }

                    for (let j = 0; j < 7; j++) {
                        let cell = row[this.showWeekNumber ? j + 1 : j];

                        if(!cell) {
                            cell = { row: i, column: j, type: 'normal', inRange: false, start: false, end: false };
                        }

                        cell.type = 'normal';

                        const index = i * 7 + j;
                        const time = nextDate(startDate, index - offset).getTime();

                        /**     仅选择一次时 选择状态不可保持     **/
                        if(!this.maxDate) {
                            // cell.inRange = time >= clearHours(this.minDate);
                            cell.inRange = cell.end = cell.start = this.minDate &&
                                                                    time === clearHours(this.minDate);
                        } else {
                            cell.inRange = time >= clearHours(this.minDate) &&
                                        time <= clearHours(this.maxDate);
                            cell.start = this.minDate && time === clearHours(this.minDate);
                            cell.end = this.maxDate && time === clearHours(this.maxDate);
                        }

                        // cell.inRange = time >= clearHours(this.minDate) &&
                        //                 time <= clearHours(this.maxDate);
                        // cell.start = this.minDate && time === clearHours(this.minDate);
                        // cell.end = this.maxDate && time === clearHours(this.maxDate);
                        /***********************************/

                        const isToday = time === now;

                        if(isToday) {
                            cell.type = 'today';
                        }

                        if(i >= 0 && i <= 1) {
                            if(j + i * 7 >= (day + offset)) {
                                cell.text = count++;
                                if(count === 2) {
                                    firstDayPosition = i * 7 + j;
                                }
                            } else {
                                cell.text = dateCountOfLastMonth - (day + offset - j % 7) + 1 + i * 7;
                                cell.type = 'prev-month';
                            }
                        } else if(count <= dateCountOfMonth) {
                            cell.text = count++;
                            if(count === 2) {
                                firstDayPosition = i * 7 + j;
                            }
                        } else {
                            cell.text = count++ - dateCountOfMonth;
                            cell.type = 'next-month';
                        }

                        let newDate = new Date(time);

                        cell.disabled = typeof disabledDate === 'function' && disabledDate(newDate);
                        /*eslint-disable*/
                        cell.selected = Array.isArray(selectedDate) &&
                          selectedDate.filter(date => date.toString() === newDate.toString())[0];
                        /*eslint-disable*/

                        this.$set(row, this.showWeekNumber ? j + 1 : j, cell);
                    }
                }

                rows.firstDayPosition = firstDayPosition;

                return rows;
            },
        },

        watch: {
            'rangeState.endDate'(newVal) {
                this.markRange(newVal);
            },

            minDate(newVal, oldVal) {
                if(newVal && newVal !== oldVal) {
                    this.rangeState.selecting = true;
                    this.markRange(newVal);
                } else if(!newVal) {
                    this.rangeState.selecting = false;
                    this.markRange(newVal);
                } else {
                    this.markRange();
                }
            },

            maxDate(newVal, oldVal) {
                if(newVal && newVal !== oldVal) {
                    this.rangeState.selecting = false;
                    this.markRange(newVal);
                    this.$emit('pick', {
                        minDate: this.minDate,
                        maxDate: this.maxDate,
                    });
                }
            },
        },

        data() {
            return {
                tableRows: [[], [], [], [], [], []],
            };
        },

        methods: {
            getCellClasses(cell) {
                const selectionMode = this.selectionMode;
                let classes = [];

                if((cell.type === 'normal' || cell.type === 'today') && !cell.disabled) {
                    classes.push('available');
                    if(cell.type === 'today') {
                        classes.push('today');
                    }
                } else {
                    classes.push(cell.type);
                }

                if(cell.disabled) {
                    classes.push('disabled');
                } else if(cell.inRange) {
                    classes.push('in-range');

                    if(cell.start) {
                        classes.push('start-date');
                    }

                    if(cell.end) {
                        classes.push('end-date');
                    }
                }

                if(cell.selected) {
                    classes.push('selected');
                }

                return classes.join(' ');
            },

            getDateOfCell(row, column) {
                const offsetFromStart = row * 7
                                        + (column - (this.showWeekNumber ? 1 : 0))
                                        - this.offsetDay;

                return nextDate(this.startDate, offsetFromStart);
            },

            markRange(maxDate) {
                const startDate = this.startDate;

                if(!maxDate) {
                    maxDate = this.maxDate;
                }

                const rows = this.rows;
                const minDate = this.minDate;

                for (let i = 0, k = rows.length; i < k; i++) {
                    const row = rows[i];

                    for (let j = 0, l = row.length; j < l; j++) {
                        /*eslint-disable*/
                        if(this.showWeekNumber && j === 0) continue;
                        /*eslint-disable*/

                        const cell = row[j];
                        const index = i * 7 + j + (this.showWeekNumber ? -1 : 0);
                        const time = nextDate(startDate, index - this.offsetDay).getTime();

                        if(maxDate && maxDate < minDate) {
                            cell.inRange = minDate &&
                                            time >= clearHours(maxDate) &&
                                            time <= clearHours(minDate);
                            cell.start = maxDate && time === clearHours(maxDate.getTime());
                            cell.end = minDate && time === clearHours(minDate.getTime());
                        } else {
                            cell.inRange = minDate &&
                                            time >= clearHours(minDate) &&
                                            time <= clearHours(maxDate);
                            cell.start = minDate && time === clearHours(minDate.getTime());
                            cell.end = maxDate && time === clearHours(maxDate.getTime());
                        }
                    }
                }
            },

            handleMouseMove(event) {
                if(!this.rangeState.selecting) {
                    return;
                }

                this.$emit('changerange', {
                    minDate: this.minDate,
                    maxDate: this.maxDate,
                    rangeState: this.rangeState,
                });

                let target = event.target;

                if(target.tagName === 'SPAN') {
                    target = target.parentNode.parentNode;
                }
                if(target.tagName === 'DIV') {
                    target = target.parentNode;
                }
                if(target.tagName !== 'TD') return;
                else if(hasClass(target, 'disabled')) return;

                const column = target.cellIndex;
                const row = target.parentNode.rowIndex - 1;
                const { row: oldRow, column: oldColumn } = this.rangeState;

                if(oldRow !== row || oldColumn !== column) {
                    this.rangeState.row = row;
                    this.rangeState.column = column;

                    this.rangeState.endDate = this.getDateOfCell(row, column);
                }
            },

            handleClick(event) {
                let target = event.target;

                if(target.tagName === 'SPAN') {
                    target = target.parentNode.parentNode;
                }
                if(target.tagName === 'DIV') {
                    target = target.parentNode;
                }

                if(target.tagName !== 'TD') return;
                if(hasClass(target, 'disabled')) {
                    return;
                }

                const selectionMode = this.selectionMode;

                let year = Number(this.year);
                let month = Number(this.month);

                const cellIndex = target.cellIndex;
                const rowIndex = target.parentNode.rowIndex;
                const cell = this.rows[rowIndex - 1][cellIndex];
                const text = cell.text;
                const className = target.className;

                const newDate = new Date(year, month, 1);

                if(className.indexOf('prev') !== -1) {
                    if(month === 0) {
                        year -= 1;
                        month = 11;
                    } else {
                        month -= 1;
                    }
                    newDate.setFullYear(year);
                    newDate.setMonth(month);
                } else if(className.indexOf('next') !== -1) {
                    if(month === 11) {
                        year += 1;
                        month = 0;
                    } else {
                        month += 1;
                    }
                    newDate.setFullYear(year);
                    newDate.setMonth(month);
                }

                newDate.setDate(parseInt(text, 10));

                if(this.selectionMode === 'range') {
                    if(this.minDate && this.maxDate) {
                        // 最小，最大日期独有

                        const minDate = new Date(newDate.getTime());
                        const maxDate = null;

                        this.$emit('pick', { minDate, maxDate }, false);
                        this.rangeState.selecting = true;
                        this.markRange(this.minDate);
                        this.$nextTick(() => {
                            this.handleMouseMove(event);
                        });
                    } else if(this.minDate && !this.maxDate) {
                        // 有最小日期 无最大日期

                        if(newDate >= this.minDate) {
                            const maxDate = new Date(newDate.getTime());

                            this.rangeState.selecting = false;
                            this.$emit('pick', {
                                minDate: this.minDate,
                                maxDate,
                            });
                        } else {
                            const minDate = new Date(newDate.getTime());

                            this.rangeState.selecting = false;
                            this.$emit('pick', { minDate, maxDate: this.minDate });
                        }
                    } else if(!this.minDate) {
                        // 无最小日期

                        const minDate = new Date(newDate.getTime());

                        this.$emit('pick', { minDate, maxDate: this.maxDate }, false);
                        this.rangeState.selecting = true;
                        this.markRange(this.minDate);
                    }
                } else if(selectionMode === 'day') {
                    this.$emit('pick', newDate);
                }
            },
        },
    };
</script>

