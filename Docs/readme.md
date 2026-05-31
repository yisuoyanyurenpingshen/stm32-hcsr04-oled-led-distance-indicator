这里显示的是代码心得，主要的问题在下面
1.先判断状态，然后再执行，不要一边判断，一边执行！！！！！
错误示范
if (distance_cm == 999 || distance_cm > 50)
{
    LED_Show_Level(0);

}
else if (distance_cm > 30)
{
    LED_Show_Level(1);
}
else if (distance_cm > 15)
{
    LED_Show_Level(2);
}
else
{
    LED_Show_Level(3);
}

正确代码
if (distance_cm == 999 || distance_cm > 50)
    {
        led_level = 0;

    }
    else if (distance_cm > 30)
    {
       led_level = 1;
    }
    else if (distance_cm > 15)
    {
       led_level = 2;
    }
    else
    {
       led_level = 3;
    }
		LED_Show_Level(led_level);
