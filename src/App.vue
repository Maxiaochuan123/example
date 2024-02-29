<template>
	<el-form ref="ruleFormRef" :model="ruleForm" :rules="rules" class="demo-ruleForm">
		<el-form-item label="问题类型" prop="questionType">
			<el-select
				class="question-type-select"
				v-model="ruleForm.questionType"
				placeholder="请选择问题类型"
			>
				<el-option v-for="item in QuestionOptions" :label="item.label" :value="item.value" />
			</el-select>
		</el-form-item>
		<el-form-item label="问题题目" required>
			<LanguageControl
				class="language-control"
				ref="questionTitleRef"
				v-model:activeName="questionTitleActiveName"
				v-model:ruleForm="ruleForm.questionTitle"
				placeholder="请输入问题题目"
			/>
		</el-form-item>
		<el-form-item
			class="question-options"
			label="问题选项"
			required
			:error="questionOptionsErrorMsg"
			:show-message="questionOptionsErrorState"
		>
			<div :class="{ 'mb-20': questionOptionsVisible }">
				<el-button
					type="success"
					v-show="addQuestionOptionsBtnVisible"
					@click="changeQuestionOptionsVisible(true)"
					>添加选项</el-button
				>
			</div>
			<div
				:class="[
					'question-options-value',
					{ 'mb-20': questionOptionsLanguageControl_Mb20_Visible }
				]"
			>
				<div class="items" v-for="(item, index) in ruleForm.questionOptions">
					<div class="value">{{ item.value }}</div>
					<div class="operating-button">
						<el-button type="primary" @click="editQuestionOptions(item, index)">编辑</el-button>
						<el-button type="danger" @click="removeQuestionOptions(item, index)">删除</el-button>
					</div>
				</div>
			</div>
			<LanguageControl
				class="language-control"
				v-if="questionOptionsVisible"
				ref="questionOptionsRef"
				v-model:activeName="questionOptionsActiveName"
				v-model:ruleForm="ruleForm.questionOptions"
				placeholder="请输入问题选项"
			>
				<div class="question-options_operating-button">
					<el-button type="primary" size="small" @click="questionOptionsConfirm">确定</el-button>
					<el-button size="small" @click="changeQuestionOptionsVisible(false)">取消</el-button>
				</div>
			</LanguageControl>
		</el-form-item>

		<el-divider />

		<div class="submit-bar">
			<el-button type="primary" @click="submitForm">提交</el-button>
			<el-button @click="cancelSubmit">取消</el-button>
		</div>
	</el-form>
</template>

<script setup>
	import { ref, reactive, computed, nextTick } from 'vue'
	import { QuestionOptions, LanguageOptions } from './enum/app'
	import { ElMessage } from 'element-plus'
	import LanguageControl from './components/LanguageControl.vue'

	let ruleForm = reactive({
		questionType: '',
		questionTitle: {
			zh: null,
			en: null,
			id: null,
			'es-mx': null
		},
		questionOptions: []
	})
	const rules = reactive({
		questionType: [{ required: true, message: '请选择问题类型', trigger: 'change' }]
	})

	let questionTitleActiveName = ref('zh')
	let questionOptionsActiveName = ref('zh')

	/**
	 * 问题选项
	 */
	let questionOptionsErrorMsg = ref(null)
	let questionOptionsVisible = ref(false)

	const changeQuestionOptionsVisible = (value) => {
		questionOptionsVisible.value = value
	}

	// 语言控件底边距 Mb20 Visible
	const questionOptionsLanguageControl_Mb20_Visible = computed(() => {
		return questionOptionsVisible && ruleForm.questionOptions?.length >= 1
	})

	// 错误状态
	const questionOptionsErrorState = computed(() => {
		if (isSubmit.value && ruleForm.questionOptions?.length < LanguageOptions.length) {
			questionOptionsErrorMsg.value =
				'选项不能为空，请点击添加按钮填写并点击确定按钮以添加问题选项，所有语言都要填写并添加'
			return true
		} else {
			questionOptionsErrorMsg.value = null
			return false
		}
	})

	// 添加按钮显示隐藏
	const addQuestionOptionsBtnVisible = computed(() => {
		return ruleForm.questionOptions?.length === 0 || !questionOptionsVisible.value ? true : false
	})

	// 数据格式化 - 取出 key 相等的当前项数据
	const questionOptionsDataFormatter = (data) => {
		let result = {}
		Object.keys(data).forEach((key) => {
			if (questionOptionsActiveName.value === key && data[key]) {
				result[key] = data[key]
			}
		})

		return result
	}

	// 确认后向 questionOptions push
	const questionOptionsConfirm = async () => {
		const data = questionOptionsRef.value.getRuleFormData()
		const newData = questionOptionsDataFormatter(data)

		const { key, value } = questionOptionItemFormatter(newData)
		if (!JSON.stringify(ruleForm.questionOptions).includes(questionOptionsActiveName.value)) {
			if (Object.keys(newData).length && !JSON.stringify(ruleForm.questionOptions).includes(key)) {
				ruleForm.questionOptions.push({
					key,
					value
				})
			}

			questionOptionsErrorMsg.value = null
		} else {
			let editIndex = null
			ruleForm.questionOptions.forEach((item, index) => {
				if (item.key === questionOptionsActiveName.value) {
					editIndex = index
				}
			})
			ruleForm.questionOptions[editIndex].value = questionOptionsRef.value.getItem(
				questionOptionsActiveName.value
			)
		}

		await questionOptionsRef.value.validate()
	}

	// item 数据格式化-取出 key value
	const questionOptionItemFormatter = (item) => {
		const key = Object.keys(item)
		const value = Object.values(item)
		return { key: key[0], value: value[0] }
	}

	// 获取 item 的 value
	const getQuestionOptionValue = (item) => {
		const { key } = questionOptionItemFormatter(item)
		return item[key]
	}

	// 编辑
	const editQuestionOptions = (item) => {
		questionOptionsRef.value.setItem(item.key, item.value)
	}

	// 移除
	const removeQuestionOptions = (item, index) => {
		ruleForm.questionOptions.splice(index, 1)
		questionOptionsRef.value.clearItemValue(item.key)
	}

	/**
	 * submit
	 */
	let isSubmit = ref(false)
	const ruleFormRef = ref(null)
	const questionTitleRef = ref(null)
	const questionOptionsRef = ref(null)

	// languageControl 语言控件校验
	const languageControlValid = () => {
		const languageControlValidList = [
			questionTitleRef.value?.validate(),
			questionOptionsRef.value?.validate()
		]
		return Promise.all(languageControlValidList)
	}

	const submitForm = () => {
		if (!ruleFormRef.value) return
		isSubmit.value = true
		ruleFormRef.value.validate(async (valid) => {
			await nextTick()
			await languageControlValid()

			if (valid && !questionOptionsErrorState.value) {
				console.log('submit!', ruleForm)
				openSubmitMessageBox()
				isSubmit.value = false
			} else {
				console.log('error submit!')
				isSubmit.value = true
			}
		})
	}

	// cancelSubmit
	const cancelSubmit = () => {
		isSubmit.value = false
		ruleForm.questionOptions = []
		ruleFormRef.value.resetFields()
		questionTitleRef.value.ruleFormReset()
		questionOptionsRef.value.ruleFormReset()
	}

	// message
	const openSubmitMessageBox = () => {
		ElMessage({
			message: '校验成功! 数据提交数据已打印在控制台请 F12 查看',
			type: 'success'
		})
	}
</script>

<style scoped>
	.mb-20 {
		margin-bottom: 20px;
	}

	.demo-ruleForm {
		width: 800px;
		.el-form-item {
			margin-bottom: 30px;
		}
		.question-type-select {
			width: 300px;
		}
		.question-options ::v-deep .el-form-item__content {
			flex-direction: column;
			align-items: flex-start;
			align-content: start;
		}

		.question-options-value {
			width: 100%;
			.items {
				width: 100%;
				margin-bottom: 6px;
				display: flex;
				justify-content: space-between;
			}
		}

		.question-options_operating-button {
			display: flex;
			justify-content: flex-end;
		}

		.language-control {
			width: 100%;
		}
	}
</style>
