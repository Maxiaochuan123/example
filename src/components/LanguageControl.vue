<template>
	<el-form ref="ruleFormRef" :model="_ruleForm">
		<el-tabs v-model="activeName" type="border-card" @tab-click="tabClick">
			<el-tab-pane
				:label="item.label"
				:name="item.name"
				v-for="item in LanguageOptions"
				:key="item.name"
			>
				<el-form-item
					:prop="item.name"
					:rules="[
						{
							required: item.name === activeName,
							message: `${item.label}不能为空`,
							trigger: 'blur'
						}
					]"
				>
					<el-input
						:ref="(el) => setTextareaRef(el, item.name)"
						v-model="_ruleForm[item.name]"
						:placeholder="placeholder"
						type="textarea"
						:rows="5"
					/>
				</el-form-item>
			</el-tab-pane>
			<slot> </slot>
		</el-tabs>
	</el-form>
</template>

<script setup>
	import { computed, ref, reactive, nextTick } from 'vue'
	import { LanguageOptions } from '../enum/app'

	const props = defineProps({
		activeName: {
			type: String,
			default: 'zh'
		},
		ruleForm: {
			type: Object,
			default: () => ({
				zh: null,
				en: null,
				id: null,
				'es-mx': null
			})
		},
		placeholder: {
			type: String,
			default: '请输入'
		}
	})

	const emit = defineEmits(['update:activeName', 'update:ruleForm', 'getRuleFormData'])

	// ruleForm 数据格式化
	const ruleFromDataFormatter = () => {
		if (props.ruleForm.constructor === Object) {
			return props.ruleForm
		}
		if (props.ruleForm.constructor === Array) {
			return reactive({
				zh: null,
				en: null,
				id: null,
				'es-mx': null
			})
		}
	}
	let _ruleForm = ruleFromDataFormatter()

	let activeName = computed({
		get() {
			return props.activeName
		},
		set(value) {
			emit('update:activeName', value)
		}
	})

	const tabClick = () => {
		// ruleFormReset()
	}

	const ruleFormReset = () => {
		ruleFormRef.value.resetFields()
	}

	const getRuleFormData = () => {
		return _ruleForm
	}

	const getItem = (key) => {
		return _ruleForm[key]
	}

	// 设置当前 tab 值
	const setItem = (key, value) => {
		activeName.value = key
		_ruleForm[key] = value
	}

	// 清空对应 activeName 对应 _ruleForm 的值
	const clearItemValue = (key) => {
		_ruleForm[key] = null
	}

	// 获取空数据项的 key
	const getNullDataItemValidKey = () => {
		for (const key in _ruleForm) {
			if (!_ruleForm[key]) return key
		}
	}

	// textarea 获取焦点
	const textareaRefMap = ref({})

	const getTextareaRefPrefix = (key) => {
		return `textareaRef_${key}`
	}

	const setTextareaRef = (el, key) => {
		if (el) textareaRefMap.value[getTextareaRefPrefix(key)] = el
	}

	const textareaFocusValid = (key) => {
		textareaRefMap.value[getTextareaRefPrefix(key)].focus()
	}
	const switchErrorValidField = async (key) => {
		activeName.value = key
		await nextTick()

		textareaFocusValid(key)

		const validList = [ruleFormRef.value.validateField(key), validate()]
		return Promise.all(validList)
	}

	// validate
	const ruleFormRef = ref(null)

	const validate = () => {
		return new Promise((resolve, reject) => {
			ruleFormRef.value.validate(async (valid) => {
				if (valid) {
					const key = getNullDataItemValidKey()
					if (key) await switchErrorValidField(key)

					resolve(true)
				} else {
					reject(false)
				}
			})
		})
	}

	defineExpose({
		validate,
		getRuleFormData,
		setItem,
		getItem,
		clearItemValue,
		ruleFormReset
	})
</script>

<style scoped>
	.el-form-item .el-form-item {
		margin-bottom: 16px;
	}
</style>
