<template>
  <n-space style="margin-bottom: 20px;">
    <n-button type="primary" @click="showAddModal">更新信息</n-button>
  </n-space>
  <n-data-table :data="data" :columns="columns" :single-line="false"></n-data-table>
  <n-modal preset="dialog" v-model:show="showAddModalStatus" title="选课">
    <n-space vertical>
      <n-select placeholder="学生" filterable :options="students" v-model:value="form.student" />
      <n-select placeholder="课程" filterable :options="courses" v-model:value="form.course" />
      <n-select placeholder="老师" filterable :options="teachers" v-model:value="form.teacher" />
      <n-input-number placeholder="平时成绩" v-model:value="form.usual_grade" clearable />
      <n-input-number placeholder="最终成绩" v-model:value="form.final_grade" clearable />
    </n-space>
    <template #action>
      <n-button @click="addSelection">新增选课</n-button>
      <n-button @click="updateSelection">修改选课记录</n-button>
    </template>
  </n-modal>
</template>
<script setup lang="ts">
import { NSpace, NButton, NDataTable, DataTableColumns, useMessage, NInputNumber, NModal, NSelect } from 'naive-ui';
import { reactive, watch, ref, h } from 'vue'
import superagent from "superagent";
import bus from '../bus';
import type { SelectOption } from 'naive-ui';
interface Selection {
  sid: number
  name: string
  clname: string
  selections: {
    cid: number
    cname: string
    tid: number
    tnam: string
    usual_grade: number
    final_grade: number
  }[]
}
interface SelectionColumn extends Selection {
  cid: number
  cname: string
  tid: number
  tname: string
  usual_grade: number
  final_grade: number
}
interface StudentBasic {
  sid: number
  name: string
}

interface Course {
  cid: number
  cname: string
}

interface Teacher {
  tid: number
  tname: string
}

const columns: DataTableColumns<SelectionColumn> = reactive([{
  title: '学号',
  key: 'sid',
  rowSpan: (rowData) => rowData.selections.length || 1
}, {
  title: '姓名',
  key: 'name',
  rowSpan: (rowData) => rowData.selections.length || 1
}, {
  title: '课程',
  key: 'cname',
}, {
  title: '授课教师',
  key: 'tname'
}, {
  title: '平时成绩',
  key: 'usual_grade',
}, {
  title: '期末成绩',
  key: 'final_grade'
}, {
  title: '操作',
  key: 'actions',
  render(rowData) {
    return [h(NButton, {
      type: 'error',
      onClick() {
        delSelection({ sid: rowData.sid, cid: rowData.cid })
      },
      style: {
        marginRight: '10px'
      }
    }, () => '退课')]
  },
}])
const data = reactive<Selection[]>([])
const message = useMessage()
const showAddModalStatus = ref(false)
const students = reactive<SelectOption[]>([])
const courses = reactive<SelectOption[]>([])
const teachers = reactive<SelectOption[]>([])
let selections: Selection[]
const form = reactive({
  student: undefined,
  course: undefined,
  teacher: undefined,
  usual_grade: undefined,
  final_grade: undefined
})
watch(() => bus.isLogin, v => {
  if (v && !data.length) {
    getSelection()
  }
}, {
  immediate: true
})

watch(() => form.student, v => {
  console.log(v);
  const ses = selections.find(e => e.sid == v)?.selections
  if (ses) {
    courses.forEach(e => {
      if (ses.some(e1 => e.value == e1.cid)) {
        e.disabled = false
      }
    })
  } else {
    courses.forEach(e => e.disabled = false)
  }
})

watch(() => form.student, v => {
  console.log(v);
  const ses = selections.find(e => e.sid == v)?.selections
  if (ses) {
    teachers.forEach(e => {
      if (ses.some(e1 => e.value == e1.cid)) {
        e.disabled = false
      }
    })
  } else {
    teachers.forEach(e => e.disabled = false)
  }
})

function getSelection() {
  if (bus.username != 'teacher' && bus.username != 'admin' && bus.username != 'user') {
    message.error('权限不足');
    return;
  }
  superagent.get('/student/selection')
    .then(e => {
      console.log(e);
      if (e.body.code == 200) {
        data.length = 0
        selections = e.body.data
        data.push(...selections.flatMap(e => Array(e.selections.length).fill(null).map((_, i) => ({ ...e, ...e.selections[i] }))))
      } else {
        message.error('获取选课信息失败')
      }
    }).catch(e => {
      console.error(e);
      message.error('获取选课信息失败')
    })
}

function showAddModal() {
  if (bus.username != 'teacher' && bus.username != 'admin') {
    message.error('权限不足');
    return;
  }
  const loading = message.loading('加载中')
  for (const key in form) {
    if (Object.hasOwn(form, key)) {
      form[key as keyof typeof form] = undefined
    }
  }
  Promise.all([superagent.get('/student').query({
    basic: true
  }).then(e => {
    if (e.body.code == 200) {
      students.length = 0
      students.push(...(<StudentBasic[]>e.body.data).map(e => ({ label: e.name + ' ' + e.sid, value: e.sid })))
    } else {
      throw new Error("");
    }
  }), superagent.get('/course').then(e => {
    if (e.body.code == 200) {
      courses.length = 0
      courses.push(...(<Course[]>e.body.data).map(e => ({ label: e.cname, value: e.cid })))
    } else {
      throw new Error("");
    }
  }), superagent.get('/teachers').then(e => {
    if (e.body.code == 200) {
      teachers.length = 0
      teachers.push(...(<Teacher[]>e.body.data).map(e => ({ label: e.tname, value: e.tid })))
    } else {
      throw new Error("");
    }
  })]).then(() => {
    showAddModalStatus.value = true
  }).catch(e => {
    console.error(e);
    message.error('获取配置失败')
  }).finally(() => {
    loading.destroy()
  })
}
function addSelection() {
  if (bus.username != 'teacher' && bus.username != 'admin') {
    message.error('权限不足');
    return;
  }
  const loading = message.loading('加载中')

  superagent.put('/student/selection').send({
    sid: form.student,
    cid: form.course,
    tid: form.teacher,
    usual_grade: form.usual_grade,
    final_grade: form.final_grade
  }).then(e => {
    if (e.body.code == 200) {
      message.success('选课成功')
      showAddModalStatus.value = false
      getSelection()
    } else {
      message.error('选课失败')
    }
  }).catch(e => {
    console.error(e);
    message.error('选课失败')
  }).finally(() => {
    loading.destroy()
  })
}


function updateSelection() {
  if (bus.username != 'teacher' && bus.username != 'admin') {
    message.error('权限不足');
    return;
  }
  const loading = message.loading('加载中')

  superagent.put('/course/update').send({
    sid: form.student,
    cid: form.course,
    tid: form.teacher,
    usual_grade: form.usual_grade,
    final_grade: form.final_grade
  }).then(e => {
    if (e.body.code == 200) {
      message.success('修改成绩成功')
      showAddModalStatus.value = false
      getSelection()
    } else {
      message.error('修改成绩失败')
    }
  }).catch(e => {
    console.error(e);
    message.error('修改成绩失败')
  }).finally(() => {
    loading.destroy()
  })
}

function delSelection({ sid, cid }: { sid: number, cid: number }) {
  if (bus.username != 'teacher' && bus.username != 'admin') {
    message.error('权限不足');
    return;
  }
  const loading = message.loading('加载中')

  superagent.del('/student/selection').send({
    sid,
    cid
  }).then(e => {
    if (e.body.code == 200) {
      message.success('退课成功')
      showAddModalStatus.value = false
      getSelection()
    } else {
      message.error('退课失败')
    }
  }).catch(e => {
    console.error(e);
    message.error('退课失败')
  }).finally(() => {
    loading.destroy()
  })
}
</script>
<style lang="less"></style>