1. 서론
    1. 쿠버네티스는 배포와 관리 방식을 변화시켰다
        1. 정확한 리소스 예측은 성능 유지와 비용 절감에 중요한 역할
    2. LSTM은 한단계의 예측이 가능하지만 오토스케일링에 적용하기 위해서는 다단계 예측이 필요하다
    3. Multi-Step LSTM을 사용했다
        1. Single-Step 모델과 지속성 모델과 Multi-Step LSTM을 비교 평가한다.
2. 데이터 전처리
    1. 데이터 수집
        1. 쿠버네티스에 배포되어 있는 MSA 어플리케이션의 리소스 메트릭을 수집
        2. 쿠버네티스의 Metric-server와 Prometheus를 통해 수집
            1. CPU와 Memory를 30분 간격으로 2주치 수집
        3. 모델링에 사용할 특정 파드 선정
        4. 메모리 사용량과 CPU 사용량 데이터셋 분포  
        ![alt text](<Multi-Step LSTM 모델 기반 쿠버네티스 파드 리소스 예측 기법/image-20240329-085043.png>)
            1. 전체 데이터 갯수(count), 사용량 평균값(mean), 사용량 분포 최소값(min)과 최대값(max)
        5. 파드의 시간에 따른 메모리 치 CPU 사용량을 보면 하루를 주기로 계절성을 가지고 있음  
        ![alt text](<Multi-Step LSTM 모델 기반 쿠버네티스 파드 리소스 예측 기법/image-20240329-091342.png>)
        ![alt text](<Multi-Step LSTM 모델 기반 쿠버네티스 파드 리소스 예측 기법/image-20240329-091400.png>)
        ![alt text](<Multi-Step LSTM 모델 기반 쿠버네티스 파드 리소스 예측 기법/image-20240329-091414.png>)
            1. 오전 8시부터 오후시까지 CPU 사용률이 최대이며 자정을 기점으로 줄어듬
    2. 데이터 전처리
        1. Test set(3,860개), Train set(15,443개)
            1. 시계열 데이터 이므로 순차적적으로 추출(전반부 80%, 후반부 20%)
            2. Min-Max 스케일링
            3. Multi-step LSTM의 파라미터로 타임스텝을 지정함
                1. 과거 데이터를 기반으로 4개의 타임스텝 앞을 예측
    3. Baseline 모델
        1. 비교하는 모델과 비교를 위한 성능 기준(t+1 시간을 슬라이딩 하여 만들어진 지속성 모델)  
        ![alt text](<Multi-Step LSTM 모델 기반 쿠버네티스 파드 리소스 예측 기법/image-20240329-092511.png>)
    4. Multi-Step LSTM 모델  
    ![alt text](<Multi-Step LSTM 모델 기반 쿠버네티스 파드 리소스 예측 기법/image-20240329-094849.png>)
        1. 하나의 Hidden Layer를 갖는 LSTM을 사용
            1. Output Dense Layer는 타임스텝 변수와 동일한 4로 지정
        2. Loss Function: MSE
        3. Optimizer: adam
        4. One step과 Multi step으로 예측을 적용
            1. 반복 기반 방법 (t+1 예측)
            2. 직접 기반 방법(각 타임스텝마다 다른 모델을 사용)
                1. 많은 리소스 필요
        5. Epoch: 100 (사전실험을 통해 결정)
        6. Batch: 128 (사전실험을 통해 결정: RMSE)
3. 실험 및 성능 평가  
![alt text](<Multi-Step LSTM 모델 기반 쿠버네티스 파드 리소스 예측 기법/image-20240329-095817.png>)
![alt text](<Multi-Step LSTM 모델 기반 쿠버네티스 파드 리소스 예측 기법/image-20240329-095829.png>)
    1. Multi-Step LSTM 모델과 Single-Step LSTM 비교
        1. 타임스텝이 증가했는데도 정확도가 유지되었음
        2. 리소스 예측이나 파드 스케일링시에 광범위하게 활용할 수 있음에도 성능 차이가 거의 없음
    2. Baseline 모델과 비교
        1. 0.005에서 0.016가량의 오차로 우수한 정확도를 보임
    3. 메모리 사용에 대해 예측한 데이터와 실제 데이터 비교  
    ![alt text](<Multi-Step LSTM 모델 기반 쿠버네티스 파드 리소스 예측 기법/image-20240329-100328.png>)
4. 결론
    1. Multi-Step LSTM을 사용한 예측 결과는 우수한 결과가 도출되었다.(???? 정말 우수한가???)
    2. Multi-Step LSTM은 기존 예측 방법과 달리 더 넓은 타임스텝에 대해 예측을 할 수 있다는 장점
    3. 예측 모델을 사용하면 파드의 스케줄링 성능 최적화와 리소스 부족을 해결할 수 있다.
    4. 향후과제
        1. 모델링 결과를 기반으로 오토스케일링 알고리즘에 적용하는 방법론을 연구