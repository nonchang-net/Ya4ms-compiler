<template>
	<div id="app">
		<!-- <img alt="Vue logo" src="./assets/logo.png">
		<HelloWorld msg="Welcome to Your Vue.js + TypeScript App"/> -->
		<h1>Ya4ms-compiler</h1>
		<p>
			Ya4ms用アセンブラ/C like言語コンパイラ
		</p>

		<h3>アセンブリコード</h3>
<div class="textareaWrapper">
<textarea id="assembleInput">
TIA	f // テストテスト
TIY	0 ; コメントテスト
	AM // タブが無視されることをテスト
	L1: TIY	0
TIA	0
AIA	1
M-
JUMP	L2
TIY	0
MA
AO
CAL SHTS
TIY	0
TIA	1
M-
TIY	0
AM
TIA	a
CAL TIMR
JUMP	L1
L2: CAL ENDS
L3: JUMP	L3
</textarea>
</div>
		<button onfocus="this.blur();" @click="assemble()">アセンブル</button>
		<h3>出力結果</h3>
		<div class="textareaWrapper">
			<div id="assembleResult"></div>
		</div>
	</div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
// import HelloWorld from './components/HelloWorld.vue';
import Assembler from './components/Assembler';

@Component({
	components: {
		// HelloWorld,
	},
})
export default class App extends Vue {

	public assemble(): void {
		const textarea = document.getElementById('assembleInput') as HTMLTextAreaElement;
		// console.log('assemble', textarea.value);
		const assembler = new Assembler();
		try {
			assembler.assemble(textarea.value);

			let result: string = '';
			for (const code of assembler.Codes) {
				result += code.toString(16);
			}
			const resultArea = document.getElementById('assembleResult') as HTMLTextAreaElement;
			resultArea.innerHTML = result + `<br /> length : ${assembler.Codes.length}` ;

			// 開発中は正常時もdump
			console.log(`test 20190513`);
			assembler.Dump();
		} catch (e) {
			assembler.Dump();
			throw e;
		}

	}

}

</script>

<style lang="scss">

* {
	margin : 0 ;
	padding: 0 ;
	box-sizing : border-box ;
}

#app {
	font-family: 'Avenir', Helvetica, Arial, sans-serif;
	-webkit-font-smoothing: antialiased;
	-moz-osx-font-smoothing: grayscale;
	text-align: center;
	color: #2c3e50;
	margin-top: 60px;
}

h3 {
	margin: 40px 0 0;
}
ul {
	list-style-type: none;
	padding: 0;
}
li {
	display: inline-block;
	margin: 0 10px;
}
a {
	color: #42b983;
}

.textareaWrapper{
	width : 100% ;
	padding : 10px;
}

textarea {
	width : 100% ;
	height : 200px ;
	border-radius : 5px ;
	padding : 10px;
}

#assembleResult{
	width : 100% ;
	border : 1px solid gray;
	background : #eee;
	border-radius : 5px ;
	padding : 10px;
}

button {
	min-height : 50px ;
	padding : 10px ;
	font-weight : bold;
	font-size : 16px ;
	border-radius : 10px ;
	background : #eee;
}

button:hover {
	background : #fff;
}

button:active {
	background : #ccc;
}

</style>
