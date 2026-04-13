# DS部新歓イベント2026

## 目次

[CPUのコード](#cpuのコード)  
[メモリのコード](#メモリのコード)  

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
const int STATE_FETCH = 0;   // 命令取得フェーズ
const int STATE_EXECUTE = 1; // 命令実行フェーズ
int cpu_state = STATE_FETCH;
bool is_halted = false;      // システム停止フラグ

void setup() {
  Serial.begin(9600);
  pinMode(BUTTON_PIN, INPUT);
  
  // アドレスバス (出力)
  pinMode(8, OUTPUT); 
  pinMode(9, OUTPUT); 
  pinMode(10, OUTPUT); 
  pinMode(11, OUTPUT);
  // データバス (入力)
  pinMode(4, INPUT); 
  pinMode(5, INPUT); 
  pinMode(6, INPUT); 
  pinMode(7, INPUT);
  // LEDモニター (出力)
  pinMode(A0, OUTPUT); 
  pinMode(A1, OUTPUT); 
  pinMode(A2, OUTPUT); 
  pinMode(A3, OUTPUT);

  // 動作確認LED
  pinMode(13, OUTPUT);

  Serial.println("[SYSTEM] CPU Initialized.");
}

void loop() {
  // アドレスバスへ現在のPC値を出力
  digitalWrite(8, PC & 0x01);
  digitalWrite(9, (PC >> 1) & 0x01);
  digitalWrite(10, (PC >> 2) & 0x01);
  digitalWrite(11, (PC >> 3) & 0x01);

  buttonState = digitalRead(BUTTON_PIN);
  digitalWrite(13, buttonState);

  // クロック信号の立ち上がりエッジ（LOW -> HIGH）を検出
  if (buttonState == HIGH && lastButtonState == LOW && !is_halted) {
    delay(10); // データバスの安定待ち

    // データバスから値を取得
    int b0 = digitalRead(4);
    int b1 = digitalRead(5);
    int b2 = digitalRead(6); 
    int b3 = digitalRead(7);
    int data = (b3 << 3) | (b2 << 2) | (b1 << 1) | b0;

    Serial.println("-------------------------");
    Serial.print("[PC: "); Serial.print(PC);
    Serial.print("] Bus Data: "); Serial.println(data);

    // ----------------------------------------------------
    // フェーズ1: 命令取得 (Fetch)
    // ----------------------------------------------------
    if (cpu_state == STATE_FETCH) {
      IR = data; // 取得したデータを命令としてレジスタに格納
      
      if (IR == 15) { 
        is_halted = true;
        Serial.println("[EXEC] HALT: System Stopped.");
      } else {
        cpu_state = STATE_EXECUTE; // 次のクロックで実行フェーズへ移行
        Serial.print("[FETCH] Instruction Loaded: "); Serial.println(IR);
      }
    } 
    // ----------------------------------------------------
    // フェーズ2: 命令実行 (Execute)
    // ----------------------------------------------------
    else if (cpu_state == STATE_EXECUTE) {
      
      if (IR == 1) { // 命令コード 1: LOAD
        ACC = data;
        Serial.print("[EXEC] LOAD: ACC = "); 
        Serial.println(ACC);
      } 
      else if (IR == 3) { // 命令コード 3: ADD
        ACC = (ACC + data) & 0x0F; // 4ビットマスク（オーバーフロー処理）
        Serial.print("[EXEC] ADD: ACC = "); 
        Serial.println(ACC);
      }
      
      cpu_state = STATE_FETCH; // 次のクロックで再び取得フェーズへ移行
    }

    // アキュムレータ(ACC)の値をLEDに出力
    digitalWrite(A0, ACC & 0x01);
    digitalWrite(A1, (ACC >> 1) & 0x01);
    digitalWrite(A2, (ACC >> 2) & 0x01);
    digitalWrite(A3, (ACC >> 3) & 0x01);

    PC++; // プログラムカウンタをインクリメント
    delay(50); // スイッチのチャタリング防止
  }

  lastButtonState = buttonState;
}


Arduino（メモリ）のコード
```

---

# メモリのコード

```cpp

// --- Arduino B (メモリ役) ---

// 【機械語プログラムの定義】
// 命令コード： 1=LOAD(読み込み), 3=ADD(足し算), 15=HALT(停止)

int ram[16] = {
  1,  // 0番地: [命令] LOAD (次の番地のデータをレジスタに入れろ)
  3,  // 1番地: [データ] 3
  3,  // 2番地: [命令] ADD (次の番地のデータをレジスタに足せ)
  4,  // 3番地: [データ] 4
  15, // 4番地: [命令] HALT (システム停止)
  0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0 // 5〜15番地は未使用
};

void setup() {
  pinMode(8, INPUT); 
  pinMode(9, INPUT); 
  pinMode(10, INPUT); 
  pinMode(11, INPUT);
  pinMode(4, OUTPUT); 
  pinMode(5, OUTPUT); 
  pinMode(6, OUTPUT); 
  pinMode(7, OUTPUT);
}

void loop() {
  // Aからの要求番地を読み取る
  int a0 = digitalRead(8);
  int a1 = digitalRead(9);
  int a2 = digitalRead(10);
  int a3 = digitalRead(11);
  int address = (a3 << 3) | (a2 << 2) | (a1 << 1) | a0;

  // メモリからデータを取り出してAへ返す
  int data = ram[address];
  digitalWrite(4, data & 0x01);
  digitalWrite(5, (data >> 1) & 0x01);
  digitalWrite(6, (data >> 2) & 0x01);
  digitalWrite(7, (data >> 3) & 0x01);
}
```
