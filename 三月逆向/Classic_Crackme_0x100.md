逆向分析得到代码:
/* 逆向分析得到的加密函数 */
int main(void) {
    // --- 变量声明 ---
    uint uVar1;
    int iVar2;
    size_t sVar3;
    char input;
    char output;
    // ... 其他变量

    // --- 初始化操作 ---
    builtin_strncpy(output, "mpknnphjngbhgzydttvkahppevhkmpwgdzxsykkokriepfnrdm", 0x33);
    setvbuf(stdout, (char*)0x0, 2, 0);
    printf("Enter the secret password: ");
    __isoc99_scanf("%50s", input); // 修正后的格式化输入

    // --- 三重加密循环 ---
    sVar3 = strlen(output);
    for (int i = 0; i < 3; i++) {
        for (int i_1 = 0; i_1 < (int)sVar3; i_1++) {
            // 位掩码操作
            uVar1 = (i_1 % 0xff >> 1 & 0x55U) + (i_1 % 0xff & 0x55U);
            uVar1 = ((int)uVar1 >> 2 & 0x33U) + (uVar1 & 0x33);
            
            // 字符变换公式
            iVar2 = ((int)uVar1 >> 4) + input[i_1] - 0x61 + (uVar1 & 0xf);
            input[i_1] = (char)iVar2 + (char)(iVar2 / 0x1a) * -0x1a + 'a';
        }
    }

    // --- 结果验证 ---
    if (memcmp(input, output, (long)sVar3) == 0) {
        printf("SUCCESS! Flag: %s\n", "picoCTF{sample_flag}");
    } else {
        puts("FAILED!");
    }
    return 0;
}
然后写出解密脚本：
if __name__=="__main__":
    output="mpknnphjngbhgzydttvkahppevhkmpwgdzxsykkokriepfnrdm"
    input=['']*len(output)
    for i in range(3):

        for i_1 in range(len(output)):
            uvar1=(i_1 % 0xff >> 1 & 0x55) + (i_1 % 0xff & 0x55)
            uvar1 = (uvar1 >> 2 & 0x33) + (uvar1 & 0x33);
            temp=ord(output[i_1])-ord('a')
            for ivar2 in range(temp,256,26):
                input_char = ivar2-(uvar1 >> 4)-(uvar1 & 0xf)+97
                if 0<=input_char<=255:
                    input[i_1]=chr(input_char)
                    break
                else:
                    print(f"非法字符: i_1={i_1}, temp={temp}")
        output=''.join(input)
    print(''.join(input))
