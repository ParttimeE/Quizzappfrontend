<script lang="ts">
	import { Button, Card } from 'flowbite-svelte';
	import axios from 'axios';
	import { accessToken } from '../../../../config/stores';
	import { page } from '$app/stores';
	import { API_BASE_PATH } from '@config/api';
	import { Icon } from 'svelte-boxicons';
	import type { questionObject } from './types';
	import QuizCompleteModal from './QuizCompleteModal.svelte';
	import type { question } from '@config/navigation';

	const config = {
		headers: { Authorization: `Bearer ${$accessToken}` }
	};

	let i = 0;
	let group = -1;
	let showModal = false;
	let quizTitle = '';
	let answerCorrect: boolean[] = [];
	let achievedPoints: number = 0;
	let possiblePoints: number = 0;

	let quizData: Promise<questionObject> = getQuiz();

	async function getQuiz(): Promise<questionObject> {
		const quizID = $page.params.quizID;

		let data: questionObject = {
			Questions: [{ id: 0, title: '', points: 0, quiz_id: 0, answers: ['', '', '', ''] }]
		};

		try {
			await axios
				.get(`${API_BASE_PATH}/quizzes/quiz/questions?quiz_id=${quizID}`, config)
				.then((response) => {
					data = response.data;
				})
				.catch((error) => {
					console.log(error);
				});
		} catch (error) {
			console.log(error);
		}
		return data;
	}

	async function handleClick(event: Event) {
		const button = event.currentTarget as HTMLButtonElement
		const value = button.getAttribute('data-value')
		group = parseInt(value ?? "-1") 

		console.log('Question:', (await quizData).Questions.length, "Current:", i)

		if((await quizData).Questions.length - 1 <= i) {
			console.log('This is the end 🎵')
			handleEnd()
			return;
		}

		
		await checkAnswer();

		group = -1;
	}

	async function handleEnd() {

		((await quizData) as questionObject).Questions.forEach((question: question) => {
			possiblePoints += question.points;
		});

		await checkAnswer();

		await axios
			.get(
				`${API_BASE_PATH}/quizzes/quiz/bestlist/user/points?quiz_id=${$page.params.quizID}`,
				config
			)
			.then((response) => {
				achievedPoints = response.data.points;
				possiblePoints = response.data.possible;
			});

		if (group >= 0) {
			await axios
				.get(`${API_BASE_PATH}/quizzes/quiz?quiz_id=${$page.params.quizID}`, config)
				.then((response) => {
					quizTitle = response.data.title;
				})
				.catch((error) => {
					console.log(error);
				});
		}

		showModal = true;
	}

	async function checkAnswer() {
		const body = { question: (await quizData).Questions[i].id, chosen: group };

		if (group >= 0) {
			await axios
				.post(`${API_BASE_PATH}/user/check_answer`, body, config)
				.then((response) => {
					answerCorrect.splice(i, 0, response.data.output);
				})
				.catch((error) => {
					console.log(error);
				});

			i < (await quizData).Questions.length - 1 ? i++ : null;
		}
	}

	// function shuffleAnswersWithIndex(answers: string[]) {
	// 	const shuffledAnswers = [...answers];
	// 	for (let i = shuffledAnswers.length - 1; i > 0; i--) {
	// 		const j = Math.floor(Math.random() * (i + 1));
	// 		[shuffledAnswers[i], shuffledAnswers[j]] = [shuffledAnswers[j], shuffledAnswers[i]];
	// 	}

	// 	const shuffledAnswersObject: any = {};
	// 	answers.forEach((answer, index) => {
	// 		shuffledAnswersObject[index] = shuffledAnswers[index];
	// 	});

	// 	return shuffledAnswersObject;
	// }

	// onMount(async () => {
	// 	(await quizData).Questions.forEach((question) => {
	// 		console.log(shuffleAnswersWithIndex(question.answers));
	// 	});
	// });
</script>

{#await quizData then quiz}
	<div class="w-full flex items-center flex-col">
		<Card class="w-full max-w-2xl">
			<h6 class="mb-1 text-5xl font-bold tracking-tight text-gray-900 dark:text-white text-center">
				{quiz.Questions[i].title}
			</h6>
			<p class="text-center mt-1">{quiz.Questions[i].points} Punkte</p>
			<div class="grid gap-2 sm:grid-cols-2 sm:gap-2 mt-6">
				{#each quiz.Questions[i].answers as answer, index (answer)}
					<!-- <div class="rounded border border-gray-200 dark:border-gray-700">
						<Radio name="bordered" class="w-full p-4 text-xl" bind:group value={index}>
							{answer}
						</Radio>
					</div> -->
					<Button on:click={handleClick} data-value={index} size="xl" color={['blue', 'red', 'yellow','green' ][index]}>
						<!-- <Icon name='bx-' class="w-4 h-4 mr-2" /> -->
						{answer}
					</Button>
				{/each}
			</div>
			<div class="grid gap-4 sm:grid-cols-2 sm:gap-6 mt-3">
				<p class="flex justify-start items-center">{i + 1}/{quiz.Questions.length}</p>
				<!-- {#if quiz.Questions.length - 1 > i}
					<Button class="text-base" on:click={handleClick}>Nächste Frage</Button>
				{:else}
					<Button class="text-base" on:click={handleEnd}>Quiz beenden</Button>
				{/if} -->
			</div>
		</Card>
	</div>
	<QuizCompleteModal
		bind:showModal
		{achievedPoints}
		{answerCorrect}
		{possiblePoints}
		{quizTitle}
		{quiz}
	/>
{/await}
