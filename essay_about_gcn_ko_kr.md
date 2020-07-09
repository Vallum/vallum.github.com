* Graph Neural Network과 관련된 전산학도로서의 소소한 이야기.

수년전에 Graph (Convoulution) Neural Network이라는 새로운 도구가 생겼다기에 이번에 살펴볼 기회를 가져봤습니다. 
주위에 몇몇이 쓴다는 이야기도 있고.

전산학도로서 Graph 알고리즘은 비교적 누구에게나 익숙한 주제이긴 하죠. 
다만 다들 잘 알고 있다시피 그래프를 이론적으로 그다지 깊이있게 들어가지를 않습니다. 
다수의 문제가 NP hard이기도 해서 대부분 근사 알고리즘으로 유사 해법을 찾는데서 그치곤 합니다.

그러나 알고리즘은 그렇다치고, 자료구조의 표현형으로서의 Graph는 매우 광범위한 유연성을 가지고 있죠. 
사실 비정형 discrete 데이터 구조의 종착지일지도 모릅니다.

GCNN을 Kipf&Welling의 2016년 논문을 통해서 접했을 때의 난감함은 주로 Spectral Theory때문이었습니다. 
결국 선형대수를 다시 보게 초래한 원인입니다.

Graph Spectral Theory는 F. R. K. Chung교수를 통해서 많이 알려졌지만, 
사실 여기서 전산학도들은 대학수학 수준에서의 대수학에 대한 기본이 부족하다는 약점이 있습니다. 
특히 Finite Group Algebra요. 저도 여기서 대충 대충 넘어가느라 고생을 했습니다.

그 다음 큰 문제는 중요 논문중 하나인 DK Hammond의 2011년 논문(Wavelets on graphs via spectral graph theory)인데,  
Graph Spectral Theory를 wavelet이나 푸리에 도메인으로 해석하는게 어렵습니다. 
전기공학도라면 푸리에와 wavelet의 개념을 이미 잡고 있을수도 있죠. 
전산학도는 일단 이 부분을 대충 지나가야 합니다. 저는 푸리에를 알지만 wavelet은 몰라서 그래도 대충 넘어갔습니다.

그래도 그 다음에는 전산학도에게는 반가운 논문 하나가 있습니다.
Graph cut논문인 Shi and Malik IEEE 2000 논문(Normalized Cuts and Image Segmentation)입니다. 
문제 자체도 Image Segmentation이고 Jianbo Shi는 유펜 전산과 교수이고 말릭은 버클리 전기전산과 교수죠. 
뭔가 읽으면 이해가 될것 같았지만, Rayleigh Quotient 같은 선형대수적 도구가 빈번히 등장하기 때문에 선형대수에 대해서 비교적 자유롭지 않으면 쉽게 읽히지가 않습니다.

결국은 이 문제가 Graph Laplacian이라는 이해에 도달합니다. 
그런데 그것 자체는 아무런 정보가 되지 않을 수 있습니다. Graph Laplacian은 Laplace Operator의 이산 일반화에 불과하다는 앎에 도달해야 합니다.

Laplace Operator는 사실은 수치해석 도구죠. 기계전공처럼 FEM, FDM, CFD에 익숙하지 않다면 전산학도에게는 너무나 낯설은 주제입니다.

아....Graph partitioning은 전산학 문제가....아니었나요???

이쯤되면 중간에 건너뛴게 너무 많으면서 무엇인가 너무 아는게 없다는 느낌이 많이 들게 됩니다.
물론 graph 분할을 할때는 많은 다양한 알고리즘이나 머신러닝적, 통계학습적 접근법들이 있습니다. k뭐뭐, 등등.

그러나 연속함수의 이산적 근사나 백터스패이스의 고유값 문제로 들어가는건 매우 정통적인 응용수학입니다.
사실은 수학적 근거가 없으면 많은 해법은 경험에 의존하는 대증적인 민간요법이 되어버리죠.

몰랐습니다. 왜 많은 전산학 커리큘럼에서 선형대수를 필수 선수 과목으로 하는지.
근데 제 경험상 선형대수를 선수로 끝내기에는 너무 분야의 무게감이 큽니다. 
실제로는 그 무게감을 느끼기도 전에 포기하게 되고요.

Laplace Operator는...그거까지 하면 공대생이죠. 
전산학은 공대가 아니라는 이야기가 아니라. 현실적으로 소프트웨어 개발자라는 직업을 향해 가게되면, 아닌것처럼 살아가기도 하잖아요.

그런데, Graph Neural Network은 그런 종류의 타협이 잘 통하지 않습니다. 
저 정도만 이해해도 각종 논문들에서 불완전하게 접근한 개념들이 종종 눈에 띕니다.

당황스럽죠. 선형대수는 학부2학년 과목이란 말입니다.

그런데 이게 문제가 되는건, 인공신경망은 바로 이산 자료구조에 대한 연속함수적 해석법이란 것입니다.
Auto Diff로서 Back Propagation을 생각해보면 Laplace Operator와 자연스러운 이웃 관계로 보이기도 합니다.
그 범주에 GCNN이 있기도 하고요.

이쯤되면 세상이 뭐가 어떻게 돌아가는지 어지럽게 느껴지기도 합니다.
차근차근 접근하지 않으면 압도되오 포기하기가 쉽습니다.