# DS部 新歓イベント2026 — コンピュータの仕組みをちょっと知ろう 💻

DS部（Data Science部）の新入生歓迎ワークショップです。  
2台のArduinoを物理的なワイヤーで繋ぎ、自分の手で4ビットコンピュータを動かします。

---

## イベント内容

このワークショップでは、普段使っているコンピュータの中で**CPUとメモリがどうやって会話しているか**を体験します。

**ミッション一覧:**
1. SlackとArduino IDEのインストール
2. ブレッドボードとLEDを使ったチュートリアル回路の作成
3. CPUとメモリの情報伝達（バス）の仕組みを学ぶ
4. 2台のArduinoをワイヤーで繋いでバスを構築する
5. ボタンを手動で押してプログラムを1ステップずつ実行する

ボタンを1回押すごとにCPUが動き、最終的にLED4つで計算結果（例: 3 + 4 = 7）が表示されます。

---

## ファイル構成

### [新歓-コンピュータ.md](新歓-コンピュータ.md)
ワークショップ当日の進行スライド（Marp形式）です。

| スライド | 内容 |
|----------|------|
| ウォーミングアップ | LEDを光らせてブレッドボードと回路の基本を体験 |
| コンピュータの4つの部品 | CPU・メモリ・クロック・I/Oの役割を解説 |
| CPUとメモリの情報伝達 | アドレスバスとデータバスの仕組みを説明 |
| 機材構成 | Arduino A（CPU役）とArduino B（メモリ役）の紹介 |
| 配線作業① | アドレスバスとデータバスを繋ぐ配線手順 |
| 配線作業② | ブレッドボードと手動クロック（ボタン）の設置 |
| 結果表示LED | 計算結果を表示する4つのLEDの配線 |
| システム起動 | ボタンを押してプログラムを手動実行する手順 |

### [新歓-コンピュータ-写真集.md](新歓-コンピュータ-写真集.md)
イベント当日の配線作業や完成した回路の写真スライドです。

---

## セットアップ

### 1. Arduino IDEのインストール
https://www.arduino.cc/en/software/

### 2. コードの書き込み

**Arduino A（CPU役）** に下記のCPUのコードを書き込みます。  
**Arduino B（メモリ役）** に下記のメモリのコードを書き込みます。

---

## CPUのコード

```cpp
// --- Arduino A (CPUモジュール) ---

const int BUTTON_PIN = 2; // 手動クロック入力
int buttonState = LOW;
int lastButtonState = LOW;

// --- CPU内部レジスタ ---
int PC = 0;   // プログラムカウンタ: 次に読み込むメモリアドレス
int IR = 0;   // 命令レジスタ: 現在実行中の命令コード
int ACC = 0;  // アキュムレータ: 演算結果の一時記憶（LED出力用）

// --- CPUステート（状態） ---
const int STATE_FETCH = 0;
const int STATE_EXECUTE = 1;
int cpu_state = STATE_FETCH;
bool is_halted = false;

void setup() {
  Serial.begin(9600);
  pinMode(BUTTON_PIN, INPUT);
  
  // アドレスバス (出力)
  pinMode(8, OUTPUT); pinMode(9, OUTPUT); pinMode(10, OUTPUT); pinMode(11, OUTPUT);
  // データバス (入力)
  pinMode(4, INPUT);  pinMode(5, INPUT);  pinMode(6, INPUT);  pinMode(7, INPUT);
  // LEDモニター (出力)
  pinMode(A0, OUTPUT); pinMode(A1, OUTPUT); pinMode(A2, OUTPUT); pinMode(A3, OUTPUT);
  pinMode(13, OUTPUT);

  Serial.println("[SYSTEM] CPU Initialized.");
}

void loop() {
  digitalWrite(8, PC & 0x01);
  digitalWrite(9, (PC >> 1) & 0x01);
  digitalWrite(10, (PC >> 2) & 0x01);
  digitalWrite(11, (PC >> 3) & 0x01);

  buttonState = digitalRead(BUTTON_PIN);
  digitalWrite(13, buttonState);

  if (buttonState == HIGH && lastButtonState == LOW && !is_halted) {
    delay(10);

    int data = (digitalRead(7) << 3) | (digitalRead(6) << 2) | (digitalRead(5) << 1) | digitalRead(4);

    Serial.print("[PC: "); Serial.print(PC); Serial.print("] Bus Data: "); Serial.println(data);

    if (cpu_state == STATE_FETCH) {
      IR = data;
      if (IR == 15) {
        is_halted = true;
        Serial.println("[EXEC] HALT: System Stopped.");
      } else {
        cpu_state = STATE_EXECUTE;
        Serial.print("[FETCH] Instruction Loaded: "); Serial.println(IR);
      }
    } else if (cpu_state == STATE_EXECUTE) {
      if (IR == 1) {
        ACC = data;
        Serial.print("[EXEC] LOAD: ACC = "); Serial.println(ACC);
      } else if (IR == 3) {
        ACC = (ACC + data) & 0x0F;
        Serial.print("[EXEC] ADD: ACC = "); Serial.println(ACC);
      }
      cpu_state = STATE_FETCH;
    }

    digitalWrite(A0, ACC & 0x01);
    digitalWrite(A1, (ACC >> 1) & 0x01);
    digitalWrite(A2, (ACC >> 2) & 0x01);
    digitalWrite(A3, (ACC >> 3) & 0x01);

    PC++;
    delay(50);
  }

  lastButtonState = buttonState;
}
```

---

## メモリのコード

```cpp
// --- Arduino B (メモリ役) ---

// 命令コード： 1=LOAD, 3=ADD, 15=HALT
int ram[16] = {
  1,  // 0番地: [命令] LOAD
  3,  // 1番地: [データ] 3
  3,  // 2番地: [命令] ADD
  4,  // 3番地: [データ] 4
  15, // 4番地: [命令] HALT
  0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0
};

void setup() {
  pinMode(8, INPUT);  pinMode(9, INPUT);  pinMode(10, INPUT); pinMode(11, INPUT);
  pinMode(4, OUTPUT); pinMode(5, OUTPUT); pinMode(6, OUTPUT); pinMode(7, OUTPUT);
}

void loop() {
  int address = (digitalRead(11) << 3) | (digitalRead(10) << 2) | (digitalRead(9) << 1) | digitalRead(8);
  int data = ram[address];
  digitalWrite(4, data & 0x01);
  digitalWrite(5, (data >> 1) & 0x01);
  digitalWrite(6, (data >> 2) & 0x01);
  digitalWrite(7, (data >> 3) & 0x01);
}
```

---

## 発展課題

今回作ったのは **4ビットコンピュータ**（0〜15の範囲）です。

- `ram[]` の数値を書き換えると別の計算を試せます（例: 8 + 7）
- 引き算・論理演算など新しい命令コードを追加することもできます

興味を持った方は、ぜひDS部に入って一緒にものづくりしましょう！
