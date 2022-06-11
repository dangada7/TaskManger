<template>
    <div>

        <!-- topBar -->
        <div>
            <!-- group by-->
            <div class="lc_inline lc_padding_8px">
                <div>
                    Group By
                </div>
                <div>
                    <select style="padding: 8px" v-model="groupBy">
                        <option value="tasks">TaskID</option>
                        <option value="users">Assignee User</option>
                    </select>
                </div>
            </div>

            <!-- resolution -->
            <div class="lc_inline lc_padding_8px">
                <div>
                    Resolution
                </div>

                <select style="padding: 8px">
                    <option value="volvo">Days</option>
                    <option value="saab">Month</option>
                </select>
            </div>

            <!-- start date-->
            <div class="lc_inline lc_padding_8px">
                <div>
                    Start Date
                </div>
                <input style="padding: 8px" type="date" v-model="startDateStr">
            </div>

            <!-- end date-->
            <div class="lc_inline lc_padding_8px">
                <div>
                    End Date
                </div>
                <input style="padding: 8px" type="date" v-model="endDateStr">
            </div>

            <!-- btn -->
            <div class="lc_inline lc_padding_8px" style="background: gold; cursor: pointer; font-size: 20px" @click="clickExecBtn">
                Execute
            </div>
        </div>

        <!-- table -->
        <div class="lc_scrollable lc_padding_16px">

            <table>
                <!-- titles -->
                <tr >
                    <th v-if="groupBy ==='tasks'">Task</th>
                    <th v-else>Users</th>
                    <th v-for="currentDay in getCurrentDisplayDays" class="lc_padding_8px" >
                        {{currentDay}}
                    </th>
                </tr>

                <!-- body -->
                <tr v-for="tableRow in filterTableRows">

                    <!-- title -->
                    <td v-if="tableRow.rootRow" @click="clickRootRow(tableRow)">
                        <div class="lc_grid_1fr_auto lc_width_150px" >
                            <div class="lc_inline"> {{tableRow.title}} </div>
                            <div class="lc_inline" >
                                Arrow
                            </div>
                        </div>

                    </td>
                    <td v-else>
                        <div class="lc_paddingRight_8px"> {{tableRow.title}} </div>
                    </td>

                    <!-- preCol-->
                    <td v-for="i in prefixTaskCol(tableRow)">
                    </td>

                    <!-- task-->
                    <td :class="tableRow.rootRow? 'lc_bg_blue' : 'lc_bg_gold'"
                        :colspan="calcTaskColspan(tableRow)">
                        {{tableRow.bodyValue}}
                    </td>

                    <!-- suffixCol -->
                    <td v-for="i in suffixTaskCol(tableRow)">
                    </td>


                </tr>

            </table>
        </div>

    </div>
</template>

<script>

class TableRow  {

    constructor({title, rootRow, bodyValue, rootID, parentID, startDate, endingDate}) {
        this.title = title
        this.rootRow = rootRow
        this.bodyValue = bodyValue
        this.rootID = rootID
        this.parentID = parentID
        this.startDate = startDate
        this.endingDate = endingDate
    }
}

export default {
    name: 'TasksCalender',
    props: {
        users: Array,
        tasks: Array,
    },
    data(){
        return {

            // top bar
            startDateStr: "2021-01-01",
            endDateStr: '2021-01-15',
            currentDateStr: "2021-01-01",
            currentEndDateStr:  '2021-01-15',

            groupBy: 'tasks',

            tasksRows : [],
            closeRootRows : {},

            usersRows : [],
            closeRootUsers: {},

        }
    },
    computed:{
        getCurrentDisplayDays(){
            let startDate = new Date(this.currentDateStr)
            let endDate = new Date(this.currentEndDateStr)

            let ans =[]

            while(startDate <= endDate){
                ans.push(this.formatDate(startDate))
                startDate.setDate(startDate.getDate() + 1)
            }

            return ans
        },
        usersById(){
            let ans = {}
            this.users.forEach(user => {
                ans[user.UserID] = user
            })

            return ans
        },
        filterTableRows(){

            let arr = this.groupBy === 'tasks' ? this.tasksRows : this.usersRows

            return arr.filter(item => {
                let uid = this.groupBy + "_" + item.parentID
                return !this.closeRootRows[uid]
            })

        }
    },
    methods:{

        // created
        arrangeTableTasks(){

            // sortTasks
            let sortTask = this.tasks.sort((task1, task2) => {
                return task1.TaskID > task2.TaskID ? 1 : -1
            })

            // insert subTasks
            let subTasksByParentId = {}
            let rootTasks = []
            sortTask.forEach(item => {
                if(item.ParentTaskID !== 0 ){
                    if(!subTasksByParentId[item.ParentTaskID]){
                        subTasksByParentId[item.ParentTaskID] = []
                    }
                    subTasksByParentId[item.ParentTaskID].push(item)
                }else{
                    rootTasks.push(item)
                }
            })

            // aggregation
            rootTasks.forEach(rootTask =>{
                if(subTasksByParentId[rootTask.TaskID]){
                    subTasksByParentId[rootTask.TaskID].forEach(subTask => {
                        rootTask.AssigneeUserIDs += subTask.AssigneeUserIDs + ','
                        rootTask.EstimatedHours += subTask.EstimatedHours
                        rootTask.ActualHours += subTask.ActualHours
                    })
                }
            })

            // inset to tableRows
            sortTask.forEach(item => {

                this.tasksRows.push({
                    title: item.Title,
                    rootRow : item.ParentTaskID === 0,
                    bodyValue : `Estimated: ${item.EstimatedHours} | Actual:${item.ActualHours} | Assignees:${this.getAssignNames(item.AssigneeUserIDs)}`,
                    rootID : item.TaskID,
                    parentID : item.ParentTaskID ,
                    startDate : item.StartDT,
                    endingDate : item.DueDT,
                })
            })

        },
        arrangeTableUsers(){

            // insert subTasks
            let subTasksByUserId = {}
            this.tasks.forEach(task => {
                if(task.ParentTaskID !== 0 ){
                    let userIdArr = task.AssigneeUserIDs.split(",")
                    userIdArr.forEach(userId => {
                        if(!subTasksByUserId[userId]){
                            subTasksByUserId[userId] = []
                        }
                        subTasksByUserId[userId].push(task)
                    })

                }
            })

            // inset to tableRows
            this.users.forEach(user => {

                let EstimatedHours = 0
                let ActualHours = 0
                let startDate = null
                let lastDate = null
                let usersIds = ""
                for(let index in subTasksByUserId[user.UserID]){
                    let task = subTasksByUserId[user.UserID][index]
                    EstimatedHours += task.EstimatedHours
                    ActualHours += task.ActualHours

                    if(startDate == null || task.StartDT < startDate){
                        startDate = task.StartDT
                    }
                    if(startDate == null || task.DueDT > startDate){
                        lastDate = task.DueDT
                    }

                    usersIds += task.AssigneeUserIDs + ","
                }

                this.usersRows.push({
                    title: user.DisplayName,
                    rootRow : true,
                    bodyValue : `Estimated: ${EstimatedHours} | Actual:${ActualHours} | Assignees:${this.getAssignNames(usersIds)}`,
                    rootID : user.UserID,
                    parentID : 0 ,
                    startDate : startDate,
                    endingDate : lastDate,
                })

                for(let index in subTasksByUserId[user.UserID]) {
                    let task = subTasksByUserId[user.UserID][index]
                    this.usersRows.push({
                        title: task.Title,
                        rootRow : task.ParentTaskID === 0,
                        bodyValue : `Estimated: ${task.EstimatedHours} | Actual:${task.ActualHours} | Assignees:${this.getAssignNames(task.AssigneeUserIDs)}`,
                        rootID : 0,
                        parentID : user.UserID ,
                        startDate : startDate,
                        endingDate : task.DueDT,
                    })
                }

            })

        },

        // helpers
        getAssignNames(assigneeUserIDs){
            let usersIdArr = assigneeUserIDs.split(",")

            usersIdArr = usersIdArr.filter((value, index, array) => {
                return array.indexOf(value) === index && value !== '';
            });

            let ans = ''
            usersIdArr.forEach((userId, index) => {
                ans += this.usersById[userId].DisplayName

                if(index !== usersIdArr.length-1){
                    ans += ","
                }
            })
            return ans

        },
        formatDate(date){
            return `${date.getDate()}/${date.getMonth()+1}/${date.getFullYear()}`
        },
        dateFrom_DD_MM_YYYY(dateString){

            let dateParts = dateString.split("/");
            return new Date(+dateParts[2], dateParts[1] - 1, +dateParts[0]);
        },
        diffDays(date1, date2){
            const diffInMs   = this.dateFrom_DD_MM_YYYY(date1) - this.dateFrom_DD_MM_YYYY(date2)
            const diffInDays = diffInMs / (1000 * 60 * 60 * 24);

            return diffInDays
        },

        // tables cells
        prefixTaskCol(row){
            let firstDisplayDate = this.getCurrentDisplayDays[0]

            return this.diffDays(row.startDate, firstDisplayDate)


        },
        calcTaskColspan(row){

            const diffInMs2   = this.dateFrom_DD_MM_YYYY(row.endingDate) - this.dateFrom_DD_MM_YYYY(row.startDate)
            const diffInDays2 = diffInMs2 / (1000 * 60 * 60 * 24);

            return diffInDays2 + 1
        },
        suffixTaskCol(row){
            return this.getCurrentDisplayDays.length - (this.prefixTaskCol(row) + this.calcTaskColspan(row))
        },

        // actions
        clickExecBtn(){
            this.currentDateStr = this.startDateStr
            this.currentEndDateStr = this.endDateStr
        },
        clickRootRow(row){

            let uid = this.groupBy + "_" + row.rootID
            this.$set(this.closeRootRows, uid, !this.closeRootRows[uid])
        },

    },
    created() {
        this.arrangeTableTasks()
        this.arrangeTableUsers()
    }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">

.lc_inline {
    display: inline-block;
}

.lc_paddingRight_8px {
    padding: 0px 0px 0px 8px;
}

.lc_padding_8px {
    padding: 8px;
}

table{
    width: 100%;
}
table, th, td{
    border: 1px solid;
    border-collapse: collapse;
}

th,td{
    padding: 8px;
}

.lc_bg_blue{
    background: dodgerblue;
}

.lc_bg_gold{
    background: gold;
}

.lc_scrollable{
    overflow-x: scroll;
}

.lc_grid_1fr_auto{
    display: grid;
    grid-template-columns: 1fr auto;
}

.lc_width_150px{
    width:150px
}

.lc_padding_16px{
    padding: 16px;
}

</style>
