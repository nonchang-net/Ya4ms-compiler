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
/*tslint:disable: max-classes-per-file*/

import { Component, Vue } from 'vue-property-decorator';
// import HelloWorld from './components/HelloWorld.vue';

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

		} catch (e) {
			assembler.Dump();
			throw e;
		}

	}

}


// type Mnemonics = 'KA' | 'AO' | 'CH' | 'CY' | 'AM' | 'MA' |
// 'M+' | 'M-' | 'TIA' | 'AIA' | 'TIY' | 'AIY' | 'CIA' | 'CIY' | 'CAL' | 'JUMP';

// Ya4msアセンブラ
// GMC-4互換を目標に。
class Assembler {

	private readonly mnemonics: {[mnemonic: string]: number} = {
		'KA': 0,
		'AO': 1,
		'CH': 2,
		'CY': 3,
		'AM': 4,
		'MA': 5,
		'M+': 6,
		'M-': 7,
		'TIA': 8,
		'AIA': 9,
		'TIY': 0xA,
		'AIY': 0xB,
		'CIA': 0xC,
		'CIY': 0xD,
		'CAL': 0xE,
		'JUMP': 0xF,
	};

	private readonly serviceCalls: {[mnemonic: string]: number} = {
		'RSTO': 0,
		'SETR': 1,
		'RSTR': 2,
		'UNDEFINED': 3, // 外部ポート Ya4msではGMC-4に合わせて非対応とする
		'CMPL': 4,
		'CHNG': 5,
		'SHFT': 6,
		'ENDS': 7,
		'ERRS': 8,
		'SHTS': 9,
		'LONS': 0xA,
		'SUND': 0xB,
		'TIMR': 0xC,
		'DSPR': 0xD,
		'DEM-': 0xE,
		'DEM+': 0xF,
	};

	private readonly DELIMITER = '<<delimiter>>';

	private labels: {[str: string]: number} = {};
	private codes: number[] = [];
	public get Codes(): number[] { return this.codes; }
	private debugLog: any = [];

	public assemble(source: string) {
		let lines: string[] = source.split('\n');

		// コメント削除
		lines = lines.map((line) => {
			return line.split(';')[0].split('//')[0];
		});
		// console.log('コメント削除結果',lines);

		// tokenに分割＋label整形
		const tokens: string[] = [];

		let address = 0;

		for (let i = 0 ; i < lines.length ; i++) {
			// 空白続きを全てデリミタに置き換える
			lines[i] = lines[i].trim().replace(/\s+/g, this.DELIMITER);
			// console.log('デリミタ置き換え結果', lines[i]);
			const splitted = lines[i].split(this.DELIMITER);
			// console.log('デリミタ切り離し結果', splitted);

			for (const token of splitted) {

				if (token.length === 0) {
					continue;
				}

				// 末尾が「:」の場合、ラベルとしてアドレスを保存しtokenには含めない
				// console.log(`${token}の末尾記号= ${token[token.length - 1]}`);
				if (token[token.length - 1] === ':') {
					const label = token.substring(0, token.length - 1);
					this.labels[label] = address;
				} else {
					// tokenに追加
					tokens.push(token);
					address ++ ;
				}
			}
		}

		// console.log('トークン切り出し結果', tokens, this.labels);
		// return;

		// アセンブル
		for (let i = 0 ; i < tokens.length ; i++) {
			const token = tokens[i];
			switch (token) {

				// オペランドなし
				case 'KA':
				case 'AO':
				case 'CH':
				case 'CY':
				case 'AM':
				case 'MA':
				case 'M+':
				case 'M-':
				{
					this.codes.push(this.mnemonics[token]);

					this.debugLog.push({
						token,
						code: this.mnemonics[token],
						address: i,
					});

				}
				break;

				// オペランド一つ
				case 'TIA':
				case 'AIA':
				case 'TIY':
				case 'AIY':
				case 'CIA':
				case 'CIY':
				{
					this.codes.push(this.mnemonics[token]);
					if (tokens.length <= i + 1) {
						throw new Error(`token ${i} ${token} required 1 operand. end of token reached.`);
					}
					const operandToken = tokens[i + 1];
					const operand = Number(`0x${operandToken}`);
					if (isNaN(operand)) {
						throw new Error(`token ${i} ${token} required operand is NaN. (${operandToken})`);
					}
					if (operand < 0 || operand > 0xF) {
						throw new Error(`token ${i} ${token} required operand out of range 4bit. ${operandToken}`);
					}
					this.codes.push(this.mnemonics[token]);
					this.codes.push(operand);

					this.debugLog.push({
						token,
						code: this.mnemonics[token],
						operand,
						address: i,
					});

					i++;
				}
				break;

				// JUMP: オペランド1つ、数値範囲は4fまで、またはラベル
				// UNDONE: jump先は4fまでとする。実機仕様未確認
				case 'JUMP':
				{
					this.codes.push(this.mnemonics[token]);
					if (tokens.length <= i + 1) {
						throw new Error(`token ${i} ${token} required 2 operand. end of token reached.`);
					}
					const operandToken = tokens[i + 1];
					const operand = Number(`0x${operandToken}`);

					let jumpAddress = 0;

					if (isNaN(operand)) {
						// 0xXXでパースできなかった場合はラベルと見なして検索
						console.log(`jump operand isNaN ${operandToken}`);
						if (this.labels[operandToken] === undefined) {
							throw new Error(`token ${i} ${token} required undefined label. (${operandToken})`);
						}
						// ラベルの指定アドレスを格納
						jumpAddress = this.labels[operandToken];
					} else {
						// 即値の場合

						if (operand < 0 || operand > 0x4F) {
							throw new Error(`token ${i} ${token} required operand out of range (0x0 - 0x4F). ${operandToken}`);
						}
						jumpAddress = operand;
					}

					// アドレスを4bit分割
					const highAddress = Math.floor(jumpAddress / 0xF);
					const lowAddress = jumpAddress % 0xF ;
					console.log(`DEBUG: jump address: jumpAddress = ${jumpAddress.toString(16)} `
						+ `address : ${highAddress.toString(16)} - ${lowAddress.toString(16)}`);
					this.codes.push(highAddress);
					this.codes.push(lowAddress);


					this.debugLog.push({
						token,
						code: this.mnemonics[token],
						jumpAddress,
						address: i,
					});

					i++;
				}
				break;

				// サービスコール
				case 'CAL':
				{
					const operandToken = tokens[i + 1];
					if (this.serviceCalls[operandToken] === undefined) {
						throw new Error(`undefined service call found: ${token}`);
					}
					this.codes.push(this.mnemonics[token]);
					this.codes.push(this.serviceCalls[operandToken]);

					this.debugLog.push({
						token,
						code: this.mnemonics[token],
						operand: operandToken,
						address: i,
					});

					i++;
				}
				break;

				default:
					throw new Error(`undefined enemonick found: position: ${i}, token: ${token}`);
			}
		}

		console.log('アセンブル結果', this.codes);

	}

	public Dump(): void {
		console.log('code', this.codes, 'labels:', this.labels, 'debugLog', this.debugLog);
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
