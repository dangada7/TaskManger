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
        <div class="lc_scrollable" style="padding: 16px">

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
                <tr v-for="task in sortAndFilterTask">

                    <!-- title -->
                    <td v-if="task.ParentTaskID === 0" @click="clickRootTask(task)">
                        <div class="lc_grid_1fr_auto lc_width_150px" >
                            <div class="lc_inline"> {{task.Title}} </div>
                            <div class="lc_inline" >
                                Arrow
                            </div>
                        </div>

                    </td>
                    <td v-else>
                        <div class="lc_paddingRight_8px"> {{task.Title}} </div>
                    </td>

                    <!-- preCol-->
                    <td v-for="i in prefixTaskCol(task)">
                    </td>

                    <!-- task-->
                    <td :class="task.ParentTaskID === 0 ? 'lc_bg_blue' : 'lc_bg_gold'" :colspan="calcTaskColspan(task)">
                        Estimated: {{task.EstimatedHours}} | Actual:{{task.ActualHours}} | Assignees:{{getAssignNames(task)}}
                    </td>

                    <!-- suffixCol -->
                    <td v-for="i in suffixTaskCol(task)">
                    </td>

                </tr>

            </table>
        </div>


    </div>
</template>

<script>



import users from "@/assets/users.json";

export default {
    name: 'TasksCalender',
    props: {
        users: Array,
        tasks: Array,
    },
    data(){
        return {
            startDateStr: "2021-01-01",
            endDateStr: '2021-01-20',

            currentDateStr: "2021-01-01",
            currentEndDateStr:  '2021-01-20',

            sortTask :[],
            subTasksByParentId : {},

            groupBy: 'tasks',


            closeRootTasks : {}
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
        sortAndFilterTask(){
            return this.sortTask.filter(item => {
                return !this.closeRootTasks[item.ParentTaskID]
            })
        }
    },
    methods:{
        clickRootTask(task){
            this.$set(this.closeRootTasks, task.TaskID, !this.closeRootTasks[task.TaskID])
        },
        formatDate(date){
            return `${date.getDate()}/${date.getMonth()+1}/${date.getFullYear()}`
        },
        dateFrom_DD_MM_YYYY(dateString){

            let dateParts = dateString.split("/");
            return new Date(+dateParts[2], dateParts[1] - 1, +dateParts[0]);
        },
        diffDaysTwoDates(date1, date2){
            const diffInMs   = this.dateFrom_DD_MM_YYYY(date1) - this.dateFrom_DD_MM_YYYY(date2)
            const diffInDays = diffInMs / (1000 * 60 * 60 * 24);

            return diffInDays
        },

        prefixTaskCol(task){
            let firstDisplayDate = this.getCurrentDisplayDays[0]

            return this.diffDaysTwoDates(task.StartDT, firstDisplayDate)


        },
        calcTaskColspan(task){

            let lastDate = this.getCurrentDisplayDays[this.getCurrentDisplayDays.length - 1]

            // const diffInMs   = this.dateFrom_DD_MM_YYYY(task.DueDT) - this.dateFrom_DD_MM_YYYY(lastDate)
            // const diffInDays = diffInMs / (1000 * 60 * 60 * 24);

            const diffInMs2   = this.dateFrom_DD_MM_YYYY(task.DueDT) - this.dateFrom_DD_MM_YYYY(task.StartDT)
            const diffInDays2 = diffInMs2 / (1000 * 60 * 60 * 24);

            return diffInDays2 + 1
        },
        suffixTaskCol(task){
            return this.getCurrentDisplayDays.length - (this.prefixTaskCol(task) + this.calcTaskColspan(task))
        },

        clickExecBtn(){
            this.currentDateStr = this.startDateStr
            this.currentEndDateStr = this.endDateStr
        },
        getAssignNames(task){
            let usersIdArr = task.AssigneeUserIDs.split(",")


            usersIdArr = usersIdArr.filter(function (value, index, array) {
                return array.indexOf(value) === index && value != '';
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

        aggregationTasks(){

            // sortTasks
            this.sortTask = this.tasks.sort((task1, task2) => {
                return task1.TaskID > task2.TaskID ? 1 : -1
            })

            // insert subTasks
            let rootTasks = []
            this.sortTask.forEach(item => {
                if(item.ParentTaskID !== 0 ){
                    if(!this.subTasksByParentId[item.ParentTaskID]){
                        this.subTasksByParentId[item.ParentTaskID] = []
                    }
                    this.subTasksByParentId[item.ParentTaskID].push(item)
                }else{
                    rootTasks.push(item)
                }
            })

            // aggregation
            rootTasks.forEach(rootTask =>{
                if(this.subTasksByParentId[rootTask.TaskID]){
                    this.subTasksByParentId[rootTask.TaskID].forEach(subTask => {
                        rootTask.AssigneeUserIDs += subTask.AssigneeUserIDs + ','
                        rootTask.EstimatedHours += subTask.EstimatedHours
                        rootTask.ActualHours += subTask.ActualHours
                    })
                }
            })
        },

    },
    created() {

        this.aggregationTasks()


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

</style>
