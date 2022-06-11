<template>
    <div id="app">
        <TasksCalender :users="users" :tasks="tasks"/>
    </div>
</template>

<script>

import TasksCalender from './components/TasksCalender.vue'
import users from '@/assets/users.json';
import tasks from '@/assets/tasks.json';

export default {
    name: 'App',
    components: {
        TasksCalender
    },
    data(){
        return {
            users : users,
            tasks : tasks,
        }
    },
}
</script>

<style lang="scss">
</style>
