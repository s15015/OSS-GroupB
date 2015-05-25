# ぷよぷよ


include "el.h"

define MAIN_SCREEN 1

//画面用の関数のプロトタイプ
void MainScreen(void);

DDOBJ puyo;				// ぷよのビットマップ
int field[15][8];		// 画面データ
int cmb[15][8];			// 結合チェック用
bool elist[30];			// 消すリスト

///////////////////////////////////////////////////////////////////////////////
//	メイン関数

//プログラム起動後、一番最初に呼び出される関数。
int elMain("ぷよぷよ");
{
	elWindow(256, 448, FALSE);
	//このループ内で処理がされる。
	//この中をグルグルと回転することで、リアルタイムでの処理が実現。
	elLoop()
	{
　　//先ほど定義したMAIN_SCREENを、宣言したMainScreen関数にリンク。
	//これで、簡単に任意の画面を呼び出すことができる。
		elSetScreen(MAIN_SCREEN,MainScreen());
	}	
	//メイン関数終了で自動的に終了処理。
	elExitMain();
}

///////////////////////////////////////////////////////////////////////////////
//	ウインドウ生成関数

void elCreate(void)
{
	elDraw::Screen(256, 448);　//256×448ドットのウィンドウ
	// ぷよ画像読み込み
	puyo = elDraw::LoadObject("puyo.bmp");
	// フィールドをクリア
	for (int y = 0; y < 15; y++) {
		for (int x = 0; x < 8; x++) {
    //ぷよが全部埋まるまでのウインドウ動作
			if (x == 0 || x == 7 || y == 14) field[y][x] = 5;
			else field[y][x] = -1;
		}
	}
	elCallScreen(MAIN_SCREEN);
}

///////////////////////////////////////////////////////////////////////////////
//	キーボード関数

// 　何かキー入力があると、呼び出される関数。
// Windowsが何かキー入力を受け取ると、ここに押されたキーを渡す。
void elKeyboard(void)
{
	//[ESC]キーが押されている。
	// VK_ESCAPEとは、[ESC]キーの仮想キーコード。
case VK_ESCAPE:
	{
　　//プログラムを終了。
		elDraw::Exit();
		break;
	}
  //キーボード関数終了。
	elExitKeyboard();
}

///////////////////////////////////////////////////////////////////////////////
//	イベント関数

//Windowsでのイベントを受け取ると、呼び出される関数。
//Windowsのメッセージを拾いたい場合の定義。上のelKeyboard関数同様、内部で処理。
long elEvent(void)
{
//イベント終了。
	elExitEvent();
}

///////////////////////////////////////////////////////////////////////////////
//	メイン画面

// ぷよ結合チェック関数（再帰）
// 引数 x, y:調べるぷよ座標 pno:ぷよ番号 cno:結合番号
int CheckCombine(int x, int y, int pno, int cno)
{
if (field[y][x] != pno || cmb[y][x] != 0) return 0;　　　　　//最初の空（?）状態
int ret = 1;	
cmb[y][x] = cno;
if (y > 0) ret += CheckCombine(x, y - 1, pno, cno);		// 上
ret += CheckCombine(x + 1, y, pno, cno);		         // 右
ret += CheckCombine(x, y + 1, pno, cno);			// 下
ret += CheckCombine(x - 1, y, pno, cno);			// 左
return ret;
}

void MainScreen(void)
{
	int i, x, y;
	int kx1, ky1, kx2, ky2;			// 仮の座標
	bool flag;						// 汎用フラグ
	static int pnext1, pnext2;		// ネクストぷよ番号（０～４）
	static int pno1, pno2;			// 現在のぷよ番号（０～４）
	static int px1, py1, px2, py2;	// ぷよのＸ、Ｙ座標
	static int keyLeft = FREE_KEY;	// 左キー
	static int keyRight = FREE_KEY;	// 右キー
	static int keyDown = FREE_KEY;	// 下キー
	static int keySpace = FREE_KEY;	// スペースキー
	static bool overFlag = false;	// ゲームオーバーフラグ
	static int score = 0;			// 得点
	static DWORD rpt = 0;			// キーリピート時間
	static DWORD downTime = 1000;	// 1ブロック落下する時間(ms)
	static DWORD downCount;			// 落下時間までの猶予(ms)
	static enum {					// 状態
		NEXT,						// ネクストぷよ出現
		NORMAL,						// 通常
		FALL,						// ぷよ落下
		ERASE1,						// ぷよ消し前
		ERASE2,						// ぷよ消し
	} status = NEXT;
	if (elChangeScreen()) {
　　　　　pnext1 = rand() % 5; pnext2 = rand() % 5;
　　　　　　}
	elDraw::Clear();
	DWORD nowTime = timeGetTime();　　
	switch (status) {
	// ネクストぷよ出現
	case NEXT:
		px1 = 3; py1 = 1; px2 = 3; py2 = 0;　　　　　　
		pno1 = pnext1; pno2 = pnext2;
		pnext1 = rand() % 5; pnext2 = rand() % 5;
	//フィールドに落ちたぷよが下についた時に次のぷよが落下を始める。
		downCount = timeGetTime();
		if (downTime > 100) downTime -= 10;
		else downTime--;
		if (downTime < 10) downTime = 10;
		status = NORMAL;
		break;
	// 通常
	case NORMAL:
		kx1 = px1; ky1 = py1;
		kx2 = px2; ky2 = py2;
		elSystem::GetKey(VK_LEFT, &keyLeft);    //左操作
		elSystem::GetKey(VK_RIGHT, &keyRight);　//右操作
		elSystem::GetKey(VK_DOWN, &keyDown);　　//落下操作
	　	elSystem::GetKey(VK_SPACE, &keySpace);　//入れ替え操作
		flag = false;
　　　//space操作でぷよの入れ替え（回転）をしているところ
		if (keySpace == PUSH_KEY) {
　　　//右回転でうしろのぷよが今の位置から右にずれる
			if (kx2 > kx1) {　
　　　　　　　　　　kx2 = kx1; 
		ky2 = ky1 + 1;
　　　　　　　　　　　}
	//右回転でうしろのぷよが今の位置から左後ろにまわった
			else if (kx2 < kx1) {　　　　
	　kx2 = kx1; 
　　　　　ky2 = ky1 - 1;
	}
    //右回転でうしろのぷよが前のぷよの下にまわった
			else if (ky2 > ky1) {
	ky2 = ky1; 
	kx2 = kx1 - 1;
	}
	//元の位置に戻る
			else {
	ky2 = ky1; 
	kx2 = kx1 + 1;
	}
		} 
	//下キーでダウンキーでぷよの落下をする。下キーを離せば元の速度で落ちる。
	else if (keyDown == PUSH_KEY || keyDown == HOLD_KEY && nowTime - rpt > 10) {
			ky1++; ky2++; flag = true; rpt = nowTime;
		} 
　　　　//左キーで左に移動する。左キーを離せば元の速度で落ちる。
	else if (keyLeft == PUSH_KEY || keyLeft == HOLD_KEY && nowTime - rpt > 150) {
			kx1--; kx2--; rpt = nowTime;
		} 
　	//右キーで左に移動する。右キーを離せば元の速度で落ちる。
	else if (keyRight == PUSH_KEY || keyRight == HOLD_KEY && nowTime - rpt > 150) {
			kx1++; kx2++; rpt = nowTime;
		} 
	//一番下まで落ちたら積み上げたとしてまたぷよを初期の早さで落とす
	else if (nowTime - downCount > downTime) {
			ky1++; ky2++; flag = true; downCount = nowTime;
		}
　　　　//何も操作しなければそのまま落ちる。
		if (field[ky1][kx1] == -1 && field[ky2][kx2] == -1) {
			px1 = kx1; py1 = ky1;
			px2 = kx2; py2 = ky2;
		} else if (flag) {
	//NEXTぷよ落下準備
			field[py1][px1] = pno1;
			field[py2][px2] = pno2;
			status = FALL;
		}
		break;
	// ぷよ落下
	case FALL:
		Sleep(50);
		flag = false;
	//開始から一番下までの落下
		for (y = 12; y >= 0; y--) {
	//左端から右端までの操作
			for (x = 1; x < 7; x++) {
        //ぷよ積み上げた場所を一番下として再び落下を始める。
				if (field[y][x] != -1 && field[y + 1][x] == -1) {
					field[y + 1][x] = field[y][x];
					field[y][x] = -1;
					flag = true;
				}
			}
		}
　　　　//設定した色数が重なると「ぷよ消し前」の作業に移る。
		if (flag == false){     
　　　　　　　　status = ERASE1;
		break;
}
	// ぷよ消し前
	case ERASE1:
		// ぷよ結合チェック
		flag = false;
		for (y = 0; y < 15; y++) for (x = 0; x < 8; x++) cmb[y][x] = 0;
		for (i = 0; i < 30; i++) elist[i] = false;
		for (y = 13, i = 0; y >= 0; y--) {
			for (x = 1; x < 7; x++) {
		//同じぷよの結合具合（？）
				if (cmb[y][x] == 0 && field[y][x] != -1) {
					i++;
					int ret = CheckCombine(x, y, field[y][x], i);
	                               //同じぷよが４つそろうと消えてスコアアップ
					if (ret >= 4) {
						flag = true;
						elist[i] = true;
						score += ret * 10;
					}
				}
			}
		}
		if (flag) {
			// 結合ぷよがある場合は消したぷよと場所の入れ替え
			for (y = 13; y >= 0; y--) {
				for (x = 1; x < 7; x++) {
					if (elist[cmb[y][x]]) field[y][x] = 5;
				}
			}
			status = ERASE2;　　
		} else {
			// 結合ぷよがない場合はゲームオーバーチェックして次へ
			for (y = 0; y < 2; y++) {
				for (x = 1; x < 7; x++) {
					if (field[y][x] != -1) overFlag = true;
				}
			}
			status = NEXT;
		}
		break;
	// ぷよ消し
	case ERASE2:
		Sleep(500);　//一時的にフリーズ反応
　　　　//なくなって落ちた場所は消える前の座標よりぷよが下におちる。
		for (y = 13; y >= 0; y--) {
			for (x = 1; x < 7; x++) {
				if (field[y][x] == 5) field[y][x] = -1;
			}
		}
		status = FALL;
		break;
	}
	// 画面表示　（ラインは白、右はねずみ色）
	elDraw::Box(192, 0, 256, 448, RGB(150, 150, 150), RGB(150, 150, 150), 0);
	elDraw::Line(0, 64, 192, 64, RGB(255, 255, 255), 1);
	//スコア表示(青色）
	SHOW(210, 16, "SCORE");
	SHOW2(210, 32, "%d", score);
	SHOW(212, 128, "NEXT");
	for (y = 0; y < 14; y++) {
		for (x = 1; x < 7; x++) {
			if (field[y][x] != -1) {
				elDraw::Layer((x - 1) * 32, y * 32, puyo,
					field[y][x] * 32, 0, field[y][x] * 32 + 32, 32);
			}
		}
	}
	// 現在のぷよ表示 (ネクスト同様に色は赤・緑・青・赤・ピンク。4つ重なったらシルバー）
	if (status == NORMAL) {
		elDraw::Layer((px1 - 1) * 32, py1 * 32,
			puyo, pno1 * 32, 0, pno1 * 32 + 32, 32);
		elDraw::Layer((px2 - 1) * 32, py2 * 32,
			puyo, pno2 * 32, 0, pno2 * 32 + 32, 32);
	}
	// ネクストぷよ表示
	elDraw::Layer(208, 144, puyo, pnext2 * 32, 0, pnext2 * 32 + 32, 32);
	elDraw::Layer(208, 144 + 32, puyo, pnext1 * 32, 0, pnext1 * 32 + 32, 32);
	elDraw::Refresh();
	// ゲームオーバー処理。処理後、ウインドウを閉じる
	if (overFlag) {
		elSystem::Message("ゲームオーバー");
		elDraw::Exit();
	}
}
