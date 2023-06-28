参考视频[卡尔曼Kalman滤波实例讲解：MPU6050加速度计陀螺仪数据解算与融合滤波，附代码](https://www.bilibili.com/video/BV1bB4y1A7pr) 讲解的不是特别好，但也能让人一知半解了，还是有用的。

GitHub [JayeGu/kalman_mpu6050 (github.com)](https://github.com/JayeGu/kalman_mpu6050)

## Kalman.cpp

```c++
/* Copyright (C) 2012 Kristian Lauszus, TKJ Electronics. All rights reserved.
 This software may be distributed and modified under the terms of the GNU
 General Public License version 2 (GPL2) as published by the Free Software
 Foundation and appearing in the file GPL2.TXT included in the packaging of
 this file. Please note that GPL2 Section 2[b] requires that all works based
 on this software must also be made publicly available under the terms of
 the GPL2 ("Copyleft").
 Contact information
 -------------------
 Kristian Lauszus, TKJ Electronics
 Web      :  http://www.tkjelectronics.com
 e-mail   :  kristianl@tkjelectronics.com
 */

#include "Kalman.h"

Kalman::Kalman() {
    /* We will set the variables like so, these can also be tuned by the user */
    Q_angle = 0.001f;
    Q_bias = 0.003f;
    R_measure = 0.03f;

    angle = 0.0f; // Reset the angle
    bias = 0.0f; // Reset bias

    P[0][0] = 0.0f; // Since we assume that the bias is 0 and we know the starting angle (use setAngle), the error covariance matrix is set like so - see: http://en.wikipedia.org/wiki/Kalman_filter#Example_application.2C_technical
    P[0][1] = 0.0f;
    P[1][0] = 0.0f;
    P[1][1] = 0.0f;
};

// The angle should be in degrees and the rate should be in degrees per second and the delta time in seconds
float Kalman::getAngle(float newAngle, float newRate, float dt) {
    // KasBot V2  -  Kalman filter module - http://www.x-firm.com/?page_id=145
    // Modified by Kristian Lauszus
    // See my blog post for more information: http://blog.tkjelectronics.dk/2012/09/a-practical-approach-to-kalman-filter-and-how-to-implement-it

    // Discrete Kalman filter time update equations - Time Update ("Predict")
    // Update xhat - Project the state ahead
    /* Step 1 */
    rate = newRate - bias;
    angle += dt * rate;

    // Update estimation error covariance - Project the error covariance ahead
    /* Step 2 */
    P[0][0] += dt * (dt*P[1][1] - P[0][1] - P[1][0] + Q_angle);
    P[0][1] -= dt * P[1][1];
    P[1][0] -= dt * P[1][1];
    P[1][1] += Q_bias * dt;

    // Discrete Kalman filter measurement update equations - Measurement Update ("Correct")
    // Calculate Kalman gain - Compute the Kalman gain
    /* Step 4 */
    float S = P[0][0] + R_measure; // Estimate error
    /* Step 5 */
    float K[2]; // Kalman gain - This is a 2x1 vector
    K[0] = P[0][0] / S;
    K[1] = P[1][0] / S;

    // Calculate angle and bias - Update estimate with measurement zk (newAngle)
    /* Step 3 */
    float y = newAngle - angle; // Angle difference
    /* Step 6 */
    angle += K[0] * y;
    bias += K[1] * y;

    // Calculate estimation error covariance - Update the error covariance
    /* Step 7 */
    float P00_temp = P[0][0];
    float P01_temp = P[0][1];

    P[0][0] -= K[0] * P00_temp;
    P[0][1] -= K[0] * P01_temp;
    P[1][0] -= K[1] * P00_temp;
    P[1][1] -= K[1] * P01_temp;

    return angle;
};

void Kalman::setAngle(float angle) { this->angle = angle; }; // Used to set angle, this should be set as the starting angle
float Kalman::getRate() { return this->rate; }; // Return the unbiased rate

/* These are used to tune the Kalman filter */
void Kalman::setQangle(float Q_angle) { this->Q_angle = Q_angle; };
void Kalman::setQbias(float Q_bias) { this->Q_bias = Q_bias; };
void Kalman::setRmeasure(float R_measure) { this->R_measure = R_measure; };

float Kalman::getQangle() { return this->Q_angle; };
float Kalman::getQbias() { return this->Q_bias; };
float Kalman::getRmeasure() { return this->R_measure; };
```

## Kalman.h

```c
/* Copyright (C) 2012 Kristian Lauszus, TKJ Electronics. All rights reserved.
 This software may be distributed and modified under the terms of the GNU
 General Public License version 2 (GPL2) as published by the Free Software
 Foundation and appearing in the file GPL2.TXT included in the packaging of
 this file. Please note that GPL2 Section 2[b] requires that all works based
 on this software must also be made publicly available under the terms of
 the GPL2 ("Copyleft").
 Contact information
 -------------------
 Kristian Lauszus, TKJ Electronics
 Web      :  http://www.tkjelectronics.com
 e-mail   :  kristianl@tkjelectronics.com
 */

#ifndef _Kalman_h_
#define _Kalman_h_

class Kalman {
public:
    Kalman();

    // The angle should be in degrees and the rate should be in degrees per second and the delta time in seconds
    float getAngle(float newAngle, float newRate, float dt);

    void setAngle(float angle); // Used to set angle, this should be set as the starting angle
    float getRate(); // Return the unbiased rate

    /* These are used to tune the Kalman filter */
    void setQangle(float Q_angle);
    /**
     * setQbias(float Q_bias)
     * Default value (0.003f) is in Kalman.cpp. 
     * Raise this to follow input more closely,
     * lower this to smooth result of kalman filter.
     */
    void setQbias(float Q_bias);
    void setRmeasure(float R_measure);

    float getQangle();
    float getQbias();
    float getRmeasure();

private:
    /* Kalman filter variables */
    float Q_angle; // Process noise variance for the accelerometer
    float Q_bias; // Process noise variance for the gyro bias
    float R_measure; // Measurement noise variance - this is actually the variance of the measurement noise

    float angle; // The angle calculated by the Kalman filter - part of the 2x1 state vector
    float bias; // The gyro bias calculated by the Kalman filter - part of the 2x1 state vector
    float rate; // Unbiased rate calculated from the rate and the calculated bias - you have to call getAngle to update the rate

    float P[2][2]; // Error covariance matrix - This is a 2x2 matrix
};

#endif
```

## arduino

```c++
/******加速度计无法计算偏航角，陀螺仪计算偏航有漂移，需与磁力计融合******/
#include <Wire.h>
#include <Kalman.h> // 这个库函数可以在arduino里面装，不然就把上面俩作为库函数
#define RESTRICT_PITCH
#define fRad2Deg  57.295779513f //将弧度转为角度的乘数
#define fDeg2Rad  0.0174532925f
#define MPU 0x68 //MPU-6050的I2C地址
#define nValCnt 7 //一次读取寄存器的数量

#define HMC 0x1E //001 1110b(0x3C>>1), HMC5883的7位i2c地址  
#define MagnetcDeclination 0.0698131701f //所在地磁偏角，根据情况自行百度  
#define CalThreshold 2  //最大和最小值超过此值，则计算偏移值
float offsetX,offsetY,offsetZ;  //磁场偏移
float kalAngleX, kalAngleY,kalAngleZ; //横滚、俯仰、偏航角的卡尔曼融合值
float Gryoyaw; //偏航角的地磁测量值
Kalman kalmanX; // 实例化卡尔曼滤波
Kalman kalmanY;
//Kalman kalmanZ;
long timer;

void setup() {
  Serial.begin(9600); //初始化串口，指定波特率
  Wire.begin(); //初始化Wire库
  WriteMPUReg(0x6B, 0); //启动MPU6050
  //WriteMPUReg(0x6A, ReadMPUReg(0x6A)&0xDF);
  WriteMPUReg(0x37, ReadMPUReg(0x37)|0x02);  //开启mpu6050的IIC直通，连接磁场传感器
  float realVals[nValCnt];
  for(int i=0;i<500;i++)ReadAccGyr(realVals); //读出测量值
  float roll,pitch;
  GetRollPitch(realVals,&roll,&pitch);
  roll *= fRad2Deg; pitch *= fRad2Deg;
  kalmanX.setAngle(roll); // 设置初始角
  kalmanY.setAngle(pitch);
  
  //设置HMC5883模式
  Wire.beginTransmission(HMC); //开始通信
  Wire.write(0x00); //选择配置寄存器A
  Wire.write(0x70); //0111 0000b，具体配置见数据手册
  Wire.endTransmission();
  Wire.beginTransmission(HMC);
  Wire.write(0x02); //选择模式寄存器
  Wire.write(0x00); //连续测量模式:0x00,单一测量模式:0x01
  Wire.endTransmission();
  calibrateMag();  //地磁计校准
//  int x,y,z;
//  getRawData(&x,&y,&z); //获取地磁数据
//  kalmanZ.setAngle(calculateYaw(pitch,roll,x,y,z) * fRad2Deg);  //设定卡尔曼滤波初始值
  timer = micros(); //计时
}

void loop() {
  float realVals[nValCnt];
  ReadAccGyr(realVals); //读出测量值
  int x,y,z;
  getRawData(&x,&y,&z);//获取地磁数据
  
  double dt = (double)(micros() - timer) / 1000000; // 计算时间差
  timer = micros();  //更新时间
  
  float roll,pitch;
  GetRollPitch(realVals,&roll,&pitch);
  float Geoyaw = calculateYaw(pitch,roll,x,y,z);
  float gyroXrate = realVals[4] / 131.0; // 转换到角度/秒
  float gyroYrate = realVals[5] / 131.0;
  float gyroZrate = realVals[6] / 131.0;
  Gryoyaw += gyroZrate * dt; //由陀螺仪获取偏航角
  if(Gryoyaw>360)Gryoyaw=0;  //大于360则变为0
  // 解决加速度计角度在-180度和180度之间跳跃时的过渡问题
  roll *= fRad2Deg; pitch *= fRad2Deg; Geoyaw *= fRad2Deg;
  if ((roll < -90 && kalAngleX > 90) || (roll > 90 && kalAngleX < -90)) {
    kalmanX.setAngle(roll);
    kalAngleX = roll;
  } 
  else{
    kalAngleX = kalmanX.getAngle(roll, gyroXrate, dt); // 卡尔曼融合
  }
  if (abs(kalAngleX) > 90) gyroYrate = -gyroYrate; // 限制的加速度计读数
  kalAngleY = kalmanY.getAngle(pitch, gyroYrate, dt);  //对俯仰角滤波
//  kalAngleZ = kalmanZ.getAngle(Geoyaw, gyroZrate, dt); //对偏航角滤波
  
  //float temperature = realVals[3] / 340.0 + 36.53;  //计算温度
  Serial.print("accangleX:");Serial.print(roll);
  Serial.print(" kalAngleX:");Serial.print(kalAngleX);
  Serial.print(" accangleY:");Serial.print(pitch);
  Serial.print(" kalAngleY:");Serial.print(kalAngleY);
//  Serial.print(" Gryoyaw:");Serial.print(Gryoyaw);
  Serial.print(" Geoyaw:");Serial.print(Geoyaw);
////  Serial.print(" kalAngleZ:");Serial.print(kalAngleZ);
  Serial.print("\r\n");
//  int yrp[] = {kalAngleX,kalAngleY,Geoyaw};
//  Serial.print(String(yrp[0])+','+String(yrp[1])+','+String(yrp[2])+'\n');
  delay(100);
}

//根据俯仰角和偏航角修正地磁传感器
float calculateYaw(float Pitch,float Roll,int x ,int y,int z)
{
  x = x-offsetX;
  y = y-offsetY;
  z = z-offsetZ;
  float Hx,Hy;
  float Out;
//  Hx = cos(Pitch)*x+sin(Pitch)*z;
//  Hy = sin(Roll)*sin(Pitch)*x + cos(Roll)*y - sin(Roll)*cos(Pitch)*z;
//  Out = atan2(Hx,Hy);
  Hx = cos(Pitch)*x+sin(Roll)*sin(Pitch)*y-cos(Roll)*sin(Pitch)*z;
  Hy = cos(Roll)*y - sin(Roll)*z;
  Out = atan2(Hy,Hx);
  if(Out<0)Out+=2*PI;
  Out = Out + MagnetcDeclination;
  if(Out > 2*PI) Out -= 2*PI;
  return Out;
}

//从MPU6050读出加速度计三个分量、温度和三个角速度计
//保存在指定的数组中
void ReadAccGyr(float *pVals) {
  Wire.beginTransmission(MPU);
  Wire.write(0x3B);
  Wire.requestFrom(MPU, nValCnt * 2, true);
  Wire.endTransmission(true);
  for (int i = 0; i < nValCnt; ++i) {
    pVals[i] = Wire.read() << 8 | Wire.read();
  }
}

//读取地磁数据
void getRawData(int* x ,int* y,int* z)
{
  Wire.beginTransmission(HMC);
  Wire.write(0x03); //从寄存器3开始读数据
  //每轴的数据都是16位的
  Wire.requestFrom(HMC, 6);
  Wire.endTransmission();
  if(6<=Wire.available()){
    *x = Wire.read()<<8| Wire.read(); //X msb，X轴高8位
    *z = Wire.read()<<8| Wire.read(); //Z msb
    *y = Wire.read()<<8| Wire.read(); //Y msb
  }
}
//根据xy分量计算方向角
float calculateHeading(int* x ,int* y,int* z)
{
  float headingRadians = atan2((double)((*y)-offsetY),(double)((*x)-offsetX));
  // 保证数据在0-2*PI之间
  if(headingRadians < 0)
    headingRadians += 2*PI;
 
  float headingDegrees = headingRadians * fRad2Deg;
  headingDegrees += MagnetcDeclination; //磁偏角
 
  // <span style="font-family: Arial, Helvetica, sans-serif;">保证数据在0-360之间</span>
  if(headingDegrees > 360)
    headingDegrees -= 360;
 
  return headingDegrees;
}
//校正传感器
void calibrateMag()
{
  int x,y,z; //三轴数据
  int xMax, xMin, yMax, yMin, zMax, zMin;
  //初始化
  getRawData(&x,&y,&z);  
  xMax=xMin=x;
  yMax=yMin=y;
  zMax=zMin=z;
  offsetX = offsetY = offsetZ = 0;
 
  Serial.println("Starting Calibration......");
  Serial.println("Please turn your device around in 20 seconds");
 
  for(int i=0;i<200;i++)
  {
    getRawData(&x,&y,&z);
    // 计算最大值与最小值
    // 计算传感器绕X,Y,Z轴旋转时的磁场强度最大值和最小值
    if (x > xMax)
      xMax = x;
    if (x < xMin )
      xMin = x;
    if(y > yMax )
      yMax = y;
    if(y < yMin )
      yMin = y;
    if(z > zMax )
      zMax = z;
    if(z < zMin )
      zMin = z;
 
    delay(100);
 
    if(i%10 == 0)
    {
      Serial.print(xMax);
      Serial.print(" ");
      Serial.println(xMin);
    }
  }
  //计算修正量
  if(abs(xMax - xMin) > CalThreshold )
    offsetX = (xMax + xMin)/2;
  if(abs(yMax - yMin) > CalThreshold )
    offsetY = (yMax + yMin)/2;
  if(abs(zMax - zMin) > CalThreshold )
    offsetZ = (zMax +zMin)/2;
 
  Serial.print("offsetX:");
  Serial.print("");
  Serial.print(offsetX);
  Serial.print(" offsetY:");
  Serial.print("");
  Serial.print(offsetY);
  Serial.print(" offsetZ:");
  Serial.print("");
  Serial.println(offsetZ);
}


//算得Roll角。
void GetRollPitch(float *pRealVals,float* roll,float* pitch) {
#ifdef RESTRICT_PITCH
  float fNorm = sqrt(pRealVals[1] * pRealVals[1] + pRealVals[2] * pRealVals[2]);
  *pitch = atan2(-pRealVals[0],fNorm);
  *roll = atan2(pRealVals[1],pRealVals[2]);  //atan2和atan作用相同，但atan2在除数是0时也可以计算，所以尽量使用atan2
#else
  float fNorm = sqrt(pRealVals[2] * pRealVals[2] + pRealVals[0] * pRealVals[0]);
  *roll = atan2(pRealVals[1],fNorm);
  *pitch = atan2(-pRealVals[0],pRealVals[2]);
#endif
}

//向MPU6050写入一个字节的数据
//指定寄存器地址与一个字节的值
void WriteMPUReg(int nReg, unsigned char nVal) {
  Wire.beginTransmission(MPU);
  Wire.write(nReg);
  Wire.write(nVal);
  Wire.endTransmission(true);
}

//从MPU6050读出一个字节的数据
//指定寄存器地址，返回读出的值
unsigned char ReadMPUReg(int nReg) {
  Wire.beginTransmission(MPU);
  Wire.write(nReg);
  Wire.requestFrom(MPU, 1, true);
  Wire.endTransmission(true);
  return Wire.read();
}
```

