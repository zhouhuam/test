#2048小游戏
用win32开发的2048的控制台小游戏
小组成员：李国庆  周华明
##功能介绍：通过2的和数不断累加，通过键盘操作移动数字进行加和，逐渐实现和为2048.

游戏逻辑中包括数字的生成，合并，分数计算，以及游戏结束的判断。棋盘是 4 x 4的，且只需要显示数字，这可以用一个int数组来表示。游戏过程中用户移动之后生成数字，需要注意这个次序
合并的顺序是按移动的逆方向来合并，而且一次移动只会合并一次。
比如有一行是4 2 2 2那么向右划(移动)的结果是0 4 2 4向左划的结果是4 4 2 0不同的数字可以使用不同的颜色来显示方便辨识，提高玩家的游戏体验
我们可以专门写一个函数来绘制游戏界面，这样每次刷新的时候调用一次就行了。首先定义一个4x4的“棋盘”来存放场面上的数字:
const int W = 4;
const int H = 4;
int board[H][W];  //H:高  W：宽   board[2][1] 表示第二行第二个
之前提到我们可以使用不同的颜色来显示不同的数字，以提高辨识度。
定义一个颜色数组，这样对于棋盘上的数字i，color[log2(i)-1]，就能得到他的颜色代码:
const int color[] = {15,8,6,14, 4, 2,12 ,5,  13, 3,   10,  11,  1,   9, };
//分别对应 //2 4 8 16 32 64 128 256 512 1024 2048 4096 8192 16384

比如数字8，取 log2(8) - 1 = 2 ，颜色代号是6，对应黄色
然后写上设置文本颜色的函数
void Set_Color(int color)
{
    HANDLE handle = GetStdHandle(STD_OUTPUT_HANDLE);
    SetConsoleTextAttribute(handle, FOREGROUND_INTENSITY | color);
}
关于这个具体的解释可以参考我的另一篇博客 控制台输出彩色文本
接下来写display函数
void display()
{
    system("cls");  //调用cmd清屏
    for (int i = 0; i < H; i++)
    {
        for (int j = 0; j < W; j++)
        {
            if (board[i][j] == 0) printf("     ");
            else {
                Set_Color(color[ (int)log2(board[i][j])-1 ]); //根据数字设置颜色
                printf("%-5d", board[i][j]);
                Set_Color(7);
            }if (j < 3) putchar('|');
        }
        putchar('\n');
        for (int r = 0; r < 23; r++) putchar('-');
        putchar('\n');
    }
    printf("当前得分: %d",score);
}
很简单，只需要遍历一下4x4的board数组依次打印出里面的数字即可。
打印每个数字之前使用Set_Color(color[ (int)log2(board[i][j])-1 ]); 设置一下其专属颜色即可
printf("%-5d", board[i][j]); 表示左对齐的格式化输出，数字宽度为5，长度不足5的数字用空格在右边补齐，以此来保证我们用-和|画的格子是整齐的
最后显示一下当前的得分，这个score的累加规则下一节会写到。
数字的生成与合并
这是2048最核心的部分。首先我们来考虑相对简单的数字生成。
这里我们只生成2，2048经典版是10%的概率生成4，90%的概率生成2
首先我们肯定得在空白的地方生成，也就是board数组中为0的位置
遍历一遍，如果是0就n++，统计有多少个空格子。统计完之后我们只需要生成一个1~n的随机数就可以决定在哪个空格子里生成2
这里也可以把空格子的位置记录下来，就不需要第二次遍历了，但是考虑到最多只有16个格子，我这里就直接遍历了：
void newNum()
{
    int flag = 0;
    int n = 0;
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++)
            if (board[i][j] == 0)
                n++;
                
    srand((unsigned)time(NULL));
    if (n == 0) return;
    int end = rand() % n;
    int t = 0;
    for (int i = 0; i < 4 && !flag; i++)
        for (int j = 0; j < 4 && !flag; j++)
        {
            if (board[i][j] == 0)
            {
                if (t == end) {
                    board[i][j] += 2;
                    flag = 1;
                }
                else t++;
                
            }
        }
}
这里t是从0开始的，因为rand()%n返回的是0~n-1的数字，如果t从1开始那么第n个数会对不上。
因为可能出现场面上数字满了但是还没结束游戏（还有能合并的数字）的情况，所以一定得判断n是否等于0：if (n == 0) return;，不然对0取模会引发异常。

然后就是数字的合并了。
这里我为了省事引入dx和dy数组来表示纵向和横向的移动：

const int dx[4] = { -1,1,0,0 };  //依次对应上下左右
const int dy[4] = {  0,0,-1,1 };
我们可以观察一下2048合并的规律：
如果你按下右键，所有的数字向右移动，然后按从右往左的顺序合并，且一次移动不会合并多次（不会出现4 2 2移动一次就变成8）
可见我们需要反向遍历：如果是向右移动，那么从最右边开始往左找，对每个格子重复以下操作：
如果当前数字不为0，且右边没有数字可以合并但是还有空位，则移动到空位。
如果当前数字不为0，且右边有数字可以合并，则合并（当前格子置为0，合并的格子翻倍），这里做完要继续往左边走并且把上一次合并的位置设置为底部，即不能再次合并了
按照以上逻辑我们很容易写出控制合并的函数:

void move(int dir)
{
    int x=0,y=0;
    if (dy[dir]) //水平移动
    {
        for (int i = 0; i < H; i++) //遍历每一行
        {
            y = (dy[dir] == 1) ? 3 : 0;
            int j = y;
            int top = y;
            while (abs(j - y) < 3) {
                j -= dy[dir];
                if (board[i][top] == 0 &&board[i][j]!=0)
                {
                    board[i][top] = board[i][j];
                    board[i][j] = 0;
                }
                else if (board[i][top]!=0 && board[i][j]==board[i][top])
                {
                    board[i][top] *= 2;
                    score += board[i][top];
                    board[i][j] = 0;
                }
                else if (board[i][top] * board[i][j] != 0 && board[i][top] != board[i][j])
                {
                    top -= dy[dir];
                    if (j != top)
                    {
                        board[i][top] = board[i][j];
                        board[i][j] = 0;
                    }
                }
            }

        }
    
    }
    else if (dx[dir]) //垂直移动
    {
        for (int j = 0; j < W; j++) //遍历每一列
        {
            x = (dx[dir] == 1) ? 3 : 0;
            int i = x;
            int top = x;
            while (abs(i - x) < 3) {
                i -= dx[dir];
                if (board[top][j] == 0 && board[i][j] != 0)
                {
                    board[top][j] = board[i][j];
                    board[i][j] = 0;
                }
                else if (board[top][j] != 0 && board[i][j] == board[top][j])
                {
                    board[top][j] *= 2;
                    score += board[top][j];
                    board[i][j] = 0;
                }
                else if (board[top][j] * board[i][j] != 0 && board[top][j] != board[i][j])
                {
                    top -= dx[dir];
                    if (i != top)
                    {
                        board[top][j] = board[i][j];
                        board[i][j] = 0;
                    }
                }
            }

        }

    }
        
}
传入的参数dir是读取键盘按键识别来的，
y = (dy[dir] == 1) ? 3 : 0; 是找到遍历的起点，如果是向右移动则是找最右边也就是3，向左移动是0
top变量记录当前的底部，来方便数字的移动，并且能防止多次合并。
因为我使用的是dx和dy，所以垂直和水平方向的移动需要分别写出来。
也可以用一个二维数组代替dx和dy来表示四个方向的单位位移，这样就只需要写一遍。
score存储的是当前得分，积分规则是2+2=4，则加4分，加上合并出的数字的值

因为2048不需要快速而频繁的键盘操作，我们使用_getch函数即可(如果需要快速低延迟的按键读取，则可以使用GetAsyncKeyState函数，可以参考#部分完整代码中注释掉的宏函数)
为了方便我定义了一个keymap数组来储存四个方向键对应的值：

const int keymap[4] = {72,80,75,77}; //依次为上下左右
按照游戏的流程：生成数字->显示->玩家键盘按键->移动,合并数字为一个周期，不停的重复直到判定游戏结束为止即可

void play()
{
    newNum();
    display();
    while (true)
    {
        
        int ch = _getch();
        for (int i = 0; i < 4; i++)
            if (ch == keymap[i]) {
            move(i); 
            if(judge())
            	return;
            newNum();
            display();
        }
        Sleep(10);     //防止连按
    }
}
注意这里判断游戏是否结束应该在生成数字前判断

游戏结束判定
只要场面上没有空格子或者能合并的格子（相邻格子数字相等）, 那么遍历找这两种情况就行了，如果没找到就是游戏结束了。

bool judge()   //0:ok  1:gameover
{
   
    for (int i = 0; i < H; i++)
        for (int j = 0; j < W; j++)
        {
            if (board[i][j] == 0)
                return false;
            
            else if(i<3 && j<3)
                if (board[i][j] == board[i + 1][j] || board[i][j]==board[i][j+1])
                    return false;
               
        }
    return true;
}




