<template></template>

<script setup>
import OpenAI from 'openai'
import { onMounted } from 'vue';

const MOVIE_JSON = "https://raw.githubusercontent.com/theapache64/top250/master/top250.json"

const spinOpenAI = async (prompt) => {
    const openai = new OpenAI({ apiKey: import.meta.env.VITE_OPEN_AI_API_KEY, dangerouslyAllowBrowser: true })
    const completion = await openai.chat.completions.create({
        messages: [
            { role: "system", content: "You are a puzzle master. You are given a movie title and you have to create a puzzle from it." },
            { role: "user", content: prompt },
        ],
        model: 'gpt-4-1106-preview',
        response_format: { "type": "json_object" }
    })

    const jsonObject = JSON.parse(completion.choices[0].message.content);
    // const valuesArray = jsonObject.data.values

    console.log(jsonObject)
}

const generatePuzzle = async () => {
    // select movie from IMDB json
    const movies = await (await fetch(MOVIE_JSON)).json()
    const movie = movies[Math.floor(Math.random() * 100)]

    if (!movie) {
        throw new Error("Failed to get movie")
    }

    // get movie title
    const movieTitle = movie.name
    console.log(movieTitle);

    // prompt to create puzzle from movie title
    const prompt = `Create a puzzle from the movie title ${movieTitle}. First give a general puzzle question in a modern poetic style but use simple words, then provide three clues to guess the movie title. 1 easy clue and 2 hard clues. Easy clue: Hard clue 1: Hard clue 2: Return your response in perfect JSON format. make sure the response is exatly 4 lines. the key for general question and each clue should be 0 to 3.`

    // spin OpenAI
    await spinOpenAI(prompt)
}

onMounted(() => {
    generatePuzzle()
})

</script>