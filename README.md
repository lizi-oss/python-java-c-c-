# python-java-c-c-
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>
#include <conio.h>
#include <stdbool.h>

// 数据结构：障碍方块（FIXED）可合并一次，合并后变为NORMAL
typedef enum { EMPTY, NORMAL, FIXED } BlockType;
typedef struct {
    int value;
    BlockType type;
    bool merged; // 标记障碍是否已合并过（仅能合并一次）
} Block;

// 全局变量
Block map[4][4];
int score = 0;
int obstacleCard = 0;
const char* rankFile = "2048_rank.txt";
const int FIXED_TRIGGER_SCORE = 1000;
bool fixedGenerated = false;
const int RANK_REWARD_THRESHOLD = 10;
const int REWARD_CARD_NUM = 1;

// 函数声明
void InitGame();
void Update();
bool CheckWin();
bool CheckLose();
void GenerateBlock();
void GenerateFixedBlock(); // 生成可合并一次的障碍
bool MoveUp();
bool MoveLeft();
bool MoveRight();
bool MoveDown();
void SaveScore();
void ShowRank();
void GiveRankReward(int rank);
void UseObstacleCard();

// 初始化游戏
void InitGame() {
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++) {
            map[i][j].value = 0;
            map[i][j].type = EMPTY;
            map[i][j].merged = false;
        }
    score = 0;
    obstacleCard = 0;
    fixedGenerated = false;
    srand((unsigned int)time(NULL));
    GenerateBlock();
    GenerateBlock();
}

// 界面绝对对齐（无错位）
void Update() {
    system("cls");
    printf("===== 进阶版2048 ===== 分数：%d | 障碍卡：%d张\n", score, obstacleCard);
    printf("+-----+-----+-----+-----+\n");
    for (int i = 0; i < 4; i++) {
        printf("|");
        for (int j = 0; j < 4; j++) {
            if (map[i][j].type == EMPTY) {
                printf("     |"); // 空方块
            } else if (map[i][j].type == FIXED) {
                printf("[%2d] |", map[i][j].value); // 可合并障碍
            } else {
                printf(" %2d  |", map[i][j].value); // 普通方块
            }
        }
        printf("\n+-----+-----+-----+-----+\n");
    }
    printf("操作：W上 A左 S下 D右 | R重来 Q退出 | L排行 C消障\n");
    printf("说明：障碍方块可与相同数值合并一次，合并后可移动\n");
}

// 胜利判断（生成2048）
bool CheckWin() {
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++)
            if (map[i][j].value == 2048 && map[i][j].type == NORMAL)
                return true;
    return false;
}

// 失败判断
bool CheckLose() {
    // 检查空位
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++)
            if (map[i][j].type == EMPTY) return false;
    // 检查合并可能（包括障碍方块）
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++) {
            // 纵向可合并（障碍未合并过）
            if (i < 3 && map[i][j].value == map[i+1][j].value && map[i][j].value != 0) {
                if (map[i][j].type != FIXED || !map[i][j].merged) return false;
            }
            // 横向可合并（障碍未合并过）
            if (j < 3 && map[i][j].value == map[i][j+1].value && map[i][j].value != 0) {
                if (map[i][j].type != FIXED || !map[i][j].merged) return false;
            }
        }
    return true;
}

// 生成普通方块
void GenerateBlock() {
    int x, y;
    do {
        x = rand() % 4;
        y = rand() % 4;
    } while (map[x][y].type != EMPTY);
    map[x][y].value = (rand() % 10 == 0) ? 4 : 2;
    map[x][y].type = NORMAL;
    map[x][y].merged = false;
}

// 生成「可合并一次」的障碍方块（核心修改）
void GenerateFixedBlock() {
    if (fixedGenerated || score < FIXED_TRIGGER_SCORE) return;
    int x, y;
    do {
        x = rand() % 4;
        y = rand() % 4;
    } while (map[x][y].type != EMPTY);
    map[x][y].value = 2;
    map[x][y].type = FIXED;   // 标记为障碍
    map[x][y].merged = false; // 未合并过（可合并一次）
    fixedGenerated = true;
    printf("生成可合并障碍！按任意键继续...");
    _getch();
}

// 左移（核心：支持障碍合并一次）
bool MoveLeft() {
    bool changed = false;
    for (int i = 0; i < 4; i++) {
        int idx = 0;
        Block row[4] = {0};
        // 1. 收集非空方块（保留障碍位置）
        for (int j = 0; j < 4; j++) {
            if (map[i][j].type != EMPTY) {
                row[idx++] = map[i][j];
                if (j != idx-1) changed = true;
            }
        }
        // 补空
        while (idx < 4) {
            row[idx].value = 0;
            row[idx].type = EMPTY;
            row[idx].merged = false;
            idx++;
        }
        // 2. 合并逻辑（支持障碍合并一次）
        for (int j = 0; j < 3; j++) {
            if (row[j].value == 0) continue;
            // 障碍未合并过 + 与右侧数值相同 → 合并
            if (row[j].value == row[j+1].value) {
                bool canMerge = true;
                // 障碍方块仅能合并一次
                if (row[j].type == FIXED && row[j].merged) canMerge = false;
                if (row[j+1].type == FIXED && row[j+1].merged) canMerge = false;
                
                if (canMerge) {
                    row[j].value *= 2;
                    score += row[j].value;
                    row[j].type = NORMAL; // 合并后变为普通方块
                    row[j].merged = true; // 标记障碍已合并（仅一次）
                    row[j+1].value = 0;
                    row[j+1].type = EMPTY;
                    row[j+1].merged = false;
                    changed = true;
                    j++; // 跳过已合并的方块
                }
            }
        }
        // 3. 再次左移（合并后补空）
        idx = 0;
        Block newRow[4] = {0};
        for (int j = 0; j < 4; j++) {
            if (row[j].type != EMPTY) {
                newRow[idx++] = row[j];
            }
        }
        while (idx < 4) {
            newRow[idx].value = 0;
            newRow[idx].type = EMPTY;
            newRow[idx].merged = false;
            idx++;
        }
        // 4. 写回棋盘
        for (int j = 0; j < 4; j++) {
            if (map[i][j].value != newRow[j].value || map[i][j].type != newRow[j].type) {
                changed = true;
                map[i][j] = newRow[j];
            }
        }
    }
    return changed;
}

// 右移（镜像左移）
bool MoveRight() {
    // 镜像翻转
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 2; j++) {
            Block t = map[i][j];
            map[i][j] = map[i][3-j];
            map[i][3-j] = t;
        }
    bool changed = MoveLeft();
    // 翻转回来
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 2; j++) {
            Block t = map[i][j];
            map[i][j] = map[i][3-j];
            map[i][3-j] = t;
        }
    return changed;
}

// 上移（转置左移）
bool MoveUp() {
    // 转置棋盘
    Block temp[4][4];
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++)
            temp[j][i] = map[i][j];
    memcpy(map, temp, sizeof(map));
    bool changed = MoveLeft();
    // 转置回来
    memcpy(temp, map, sizeof(map));
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++)
            map[i][j] = temp[j][i];
    return changed;
}

// 下移（转置+镜像左移）
bool MoveDown() {
    Block temp[4][4];
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++)
            temp[3-j][i] = map[i][j];
    memcpy(map, temp, sizeof(map));
    bool changed = MoveLeft();
    memcpy(temp, map, sizeof(map));
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++)
            map[i][j] = temp[3-j][i];
    return changed;
}

// 保存分数
void SaveScore() {
    FILE* fp = fopen(rankFile, "a");
    if (fp != NULL) {
        fprintf(fp, "%d\n", score);
        fclose(fp);
    }
}

// 排行榜奖励
void GiveRankReward(int rank) {
    if (rank <= RANK_REWARD_THRESHOLD) {
        obstacleCard += REWARD_CARD_NUM;
        printf("恭喜前%d名！获得%d张障碍卡\n", RANK_REWARD_THRESHOLD, REWARD_CARD_NUM);
        printf("当前卡数：%d张\n", obstacleCard);
        _getch();
    }
}

// 显示排行榜
void ShowRank() {
    system("cls");
    printf("===== 2048排行榜 =====\n");
    int scores[100] = {0};
    int count = 0;
    FILE* fp = fopen(rankFile, "r");
    if (fp != NULL) {
        while (fscanf(fp, "%d", &scores[count]) != EOF) count++;
        fclose(fp);
    }
    // 冒泡排序
    for (int i = 0; i < count-1; i++)
        for (int j = 0; j < count-i-1; j++)
            if (scores[j] < scores[j+1]) {
                int t = scores[j];
                scores[j] = scores[j+1];
                scores[j+1] = t;
            }
    // 打印前10
    int userRank = -1;
    printf("排名 | 分数\n");
    printf("-----------\n");
    for (int i = 0; i < count && i < 10; i++) {
        printf("%d    | %d\n", i+1, scores[i]);
        if (scores[i] == score && userRank == -1) userRank = i+1;
    }
    if (count == 0) printf("暂无记录\n");
    // 发放奖励
    if (userRank == -1 && count < RANK_REWARD_THRESHOLD && score > 0) userRank = count + 1;
    if (userRank != -1) GiveRankReward(userRank);
    printf("按任意键返回...");
    _getch();
}

// 使用障碍消除卡（直接移除障碍）
void UseObstacleCard() {
    if (obstacleCard <= 0) {
        printf("无可用障碍卡！按任意键返回...\n");
        _getch();
        return;
    }
    // 查找第一个障碍
    int fx = -1, fy = -1;
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++)
            if (map[i][j].type == FIXED) {
                fx = i; fy = j;
                break;
            }
    if (fx == -1) {
        printf("当前无障碍！按任意键返回...\n");
        _getch();
        return;
    }
    // 移除障碍
    map[fx][fy].type = EMPTY;
    map[fx][fy].value = 0;
    map[fx][fy].merged = false;
    obstacleCard--;
    printf("已移除(%d,%d)障碍！剩余卡：%d张\n", fx+1, fy+1, obstacleCard);
    _getch();
}

// 主函数（按键响应+逻辑驱动）
int main() {
    char key;
    bool isWin = false;
    InitGame();
    while (1) {
        Update();
        
        // 胜利判断
        if (CheckWin() && !isWin) {
            printf("生成2048！是否继续？(Y/N) ");
            char choice = _getch();
            if (choice == 'N' || choice == 'n') {
                SaveScore();
                ShowRank();
                printf("游戏结束！最终分数：%d\n", score);
                system("pause");
                break;
            }
            isWin = true;
        }
        
        // 失败判断
        if (CheckLose()) {
            SaveScore();
            ShowRank();
            printf("游戏失败！最终分数：%d\n", score);
            system("pause");
            break;
        }
        
        // 按键处理
        key = _getch();
        bool changed = false;
        switch (key) {
            case 'w': case 'W': changed = MoveUp(); break;
            case 'a': case 'A': changed = MoveLeft(); break;
            case 's': case 'S': changed = MoveDown(); break;
            case 'd': case 'D': changed = MoveRight(); break;
            case 'r': case 'R': InitGame(); isWin = false; continue;
            case 'q': case 'Q': SaveScore(); ShowRank(); printf("退出游戏！\n"); system("pause"); return 0;
            case 'l': case 'L': ShowRank(); break;
            case 'c': case 'C': UseObstacleCard(); break;
            default: continue;
        }
        
        // 移动后生成新方块
        if (changed) {
            GenerateBlock();
            GenerateFixedBlock();
            fixedGenerated = false;
        }
    }
    return 0;
}