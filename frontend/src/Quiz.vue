<template>
	<div v-if="questions.length">
		<div v-if="showTitle">
			<button @click="startQuiz">スタート</button>
		</div>
		<div v-else-if="!submitted">
			<div class="progress-bar-wrapper">
				<div class="progress-label">
					問題 {{ currentIndex + 1 }} / {{ questions.length }}
				</div>
				<div class="progress-bar">
					<div class="progress-bar-inner" :style="{ width: ((currentIndex + 1) / questions.length * 100) + '%' }"></div>
				</div>
			</div>
			<div class="question-block">
				<p><strong>Q{{ currentIndex + 1 }}: {{ currentQuestion.question }}</strong></p>
				<div v-for="option in currentQuestion.options" :key="option">
					<label>
						<input type="radio" :name="'question-' + currentQuestion.id" :value="option"
							v-model="answers[currentQuestion.id]" @change="nextQuestion" required>
						{{ option }}
					</label>
				</div>
			</div>
			<button @click="prevQuestion" :disabled="currentIndex === 0">戻る</button>
		</div>
		<div v-else>
			<h2>あなたの回答</h2>
			<ul>
				<li v-for="q in questions" :key="q.id">
					Q{{ q.id }}: {{ answers[q.id] || '未回答' }}<br />
					回答までの累計時間: {{ formatTime(answerTimes[q.id] || 0) }}<br />
					戻った回数: {{ backCounts[q.id] || 0 }}
				</li>
			</ul>
			<button @click="restart">最初からやり直す</button>
			<button @click="downloadResult">結果をダウンロード</button>
		</div>
	</div>
	<div v-else>
		<p>読み込み中...</p>
	</div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';

const questions = ref([]);
const answers = ref({});
const currentIndex = ref(0);
const submitted = ref(false);
const answerTimes = ref({}); // 各問題の累計回答時間（ms）
const startTime = ref(0); // 現在の問題の開始時刻
const backCounts = ref({}); // 各問題ごとの戻った回数
const showTitle = ref(true); // タイトル画面表示

const currentQuestion = computed(() => questions.value[currentIndex.value] || {});

onMounted(async () => {
	const res = await fetch('/data/questions.json');
	questions.value = await res.json();
});

function startQuiz() {
	showTitle.value = false;
	currentIndex.value = 0;
	submitted.value = false;
	answers.value = {};
	answerTimes.value = {};
	backCounts.value = {};
	startTime.value = Date.now();
}

function nextQuestion() {
	// 経過時間を加算
	const qid = currentQuestion.value.id;
	const elapsed = Date.now() - startTime.value;
	answerTimes.value[qid] = (answerTimes.value[qid] || 0) + elapsed;

	if (currentIndex.value < questions.value.length - 1) {
		currentIndex.value++;
		startTime.value = Date.now();
	} else {
		submitted.value = true;
	}
}

function prevQuestion() {
	if (currentIndex.value > 0) {
		const prevIdx = currentIndex.value - 1;
		const prevQid = questions.value[prevIdx].id;
		backCounts.value[prevQid] = (backCounts.value[prevQid] || 0) + 1;
		currentIndex.value--;
		startTime.value = Date.now();
	}
}

function restart() {
	showTitle.value = true;
	backCounts.value = {};
	startTime.value = 0;
}

function formatTime(ms) {
	return ms + 'ミリ秒';
}

function downloadResult() {
	const result = {
		answers: answers.value,
		answerTimes: answerTimes.value,
		backCounts: backCounts.value,
		timestamp: new Date().toISOString()
	};
	const blob = new Blob([JSON.stringify(result, null, 2)], { type: 'application/json' });
	const url = URL.createObjectURL(blob);
	const a = document.createElement('a');
	a.href = url;
	a.download = 'quiz_result.json';
	document.body.appendChild(a);
	a.click();
	document.body.removeChild(a);
	URL.revokeObjectURL(url);
}
</script>

<style scoped>
.question-block {
	margin-bottom: 1.5em;
}
button {
	margin-top: 1em;
}
.progress-bar-wrapper {
	margin-bottom: 1em;
}
.progress-label {
	font-size: 0.95em;
	margin-bottom: 0.2em;
}
.progress-bar {
	width: 100%;
	height: 12px;
	background: #eee;
	border-radius: 6px;
	overflow: hidden;
}
.progress-bar-inner {
	height: 100%;
	background: #42b983;
	transition: width 0.3s;
}
</style>