604 

IEEE TRANSACTIONS ON SERVICES COMPUTING, VOL. 17, NO. 2, MARCH/APRIL 2024 

# PBScaler: A Bottleneck-Aware Autoscaling Framework for Microservice-Based Applications 

Shuaiyu Xie , Jian Wang _, Member, IEEE_ , Bing Li _, Member, IEEE_ , Zekun Zhang , Duantengchuan Li , and Patrick C. K. Hung 

**_Abstract_ —Autoscaling is critical for ensuring optimal performance and resource utilization in cloud applications with dynamic workloads. However, traditional autoscaling technologies are typically no longer applicable in microservice-based applications due to the diverse workload patterns and complex interactions between microservices. Specifically, the propagation of performance anomalies through interactions leads to a high number of abnormal microservices, making it difficult to identify the root performance bottlenecks (PBs) and formulate appropriate scaling strategies. In addition, to balance resource consumption and performance, the existing mainstream approaches based on online optimization algorithms require multiple iterations, leading to oscillation and elevating the likelihood of performance degradation. To tackle these issues, we propose PBScaler, a bottleneck-aware autoscaling framework designed to prevent performance degradation in a microservice-based application. The key insight of PBScaler is to locate the PBs. Thus, we propose TopoRank, a novel random walk algorithm based on the topological potential to reduce unnecessary scaling. By integrating TopoRank with an offline performanceaware optimization algorithm, PBScaler optimizes replica management without disrupting the online application. Comprehensive experiments demonstrate that PBScaler outperforms existing state-of-the-art approaches in mitigating performance issues while conserving resources efficiently.** 

**_Index Terms_ —Microservice, autoscaling, performance bottleneck, replica management.** 

## I. INTRODUCTION 

ITH the advancement of microservice architecture, an **W** increasing number of cloud applications are migrating from monolithic architecture to microservice architecture [1], [2], [3], [4], [5], [6]. This new architecture reduces application coupling by breaking a monolithic application into multiple microservices that communicate with each other via HTTP or RPC protocols [7]. Moreover, each microservice can be developed, deployed, and scaled independently by separate teams, enabling 

Manuscript received 16 March 2023; revised 25 December 2023; accepted 26 February 2024. Date of publication 18 March 2024; date of current version 10 April 2024. This work was supported in part by the National Key Research and Development Program of China under Grant 2022YFF0902701 and in part by the National Natural Science Foundation of China under Grant 62032016, Grant 61832014, and Grant 61972292. _(Corresponding authors: Jian Wang; Bing Li.)_ 

Shuaiyu Xie, Jian Wang, Bing Li, Zekun Zhang, and Duantengchuan Li are with the School of Computer Science, Wuhan University, Wuhan 430072, China (e-mail: theory@whu.edu.cn; jianwang@whu.edu.cn; bingli@whu.edu.cn; zekunzhang@whu.edu.cn; dtclee1222@whu.edu.cn). 

Patrick C. K. Hung is with the Faculty of Business and Information Technology, Ontario Tech University, Oshawa, ON L1G 0C5, Canada (e-mail: patrick.hung@ontariotechu.ca). 

Digital Object Identifier 10.1109/TSC.2024.3376202 



Fig. 1. Part of the invocation relationship in Online Boutique. 

rapid application development and iteration. Nevertheless, the unpredictability of external workloads and the complexity of interactions between microservices can result in performance degradation [8], [9], [10]. Cloud providers must prepare excessive resources to meet the service level objective (SLO) of application owners, which usually causes unnecessary waste of resources [11], [12]. As a result, the imbalance between satisfying SLO and minimizing resource consumption becomes a major challenge encountered by resource management in microservices. 

Microservice autoscaling refers to the capability of allocating resources elastically in response to workload variations [13]. By utilizing the elasticity property of microservices, autoscaling can mitigate the conflict between resource cost and performance. However, the autoscaling of microservices suffers from accurately scaling the performance bottleneck (PB) in a short period. Due to the complexity of communication between microservices, the performance degradation of a PB may propagate to other microservices via message passing [2], resulting in a high number of abnormal microservices at the same time. We demonstrated this by injecting burst workloads to specific microservices in Online Boutique,<sup>1</sup> an open-source microservice application developed by Google. Fig. 1 shows that the performancedegradationinthePB _Recommend_ canspreadtothe upstream microservices like _Checkout_ and _Frontend_ . To further verify the importance of accurately scaling the PB, we conducted stress testing and scaled different microservices separately. As shown in Fig. 2, abnormal microservice ( _Frontend_ ) scaling cannot alleviate SLO violations. However, when we identified and scaled the PB _Recommend_ , the performance of the microservice application improved. Unfortunately, locating PBs is usually time-consuming and can occasionally make mistakes [14]. 

In recent years, several approaches have been proposed to identify critical microservices before autoscaling. For example, 

> 1https://github.com/GoogleCloudPlatform/microservices-demo 

> 1939-1374 © 2024 IEEE. Personal use is permitted, but republication/redistribution requires IEEE permission. See https://www.ieee.org/publications/rights/index.html for more information. 

Authorized licensed use limited to: QUAID E AZAM UNIVERSITY. Downloaded on August 11,2026 at 11:27:28 UTC from IEEE Xplore.  Restrictions apply. 

605 

XIE et al.: PBScaler: A BOTTLENECK-AWARE AUTOSCALING FRAMEWORK FOR MICROSERVICE-BASED APPLICATIONS 





Fig. 3. Replica fluctuation and latency fluctuation of MicroScaler under burst workloads. Excessive online attempts (a) cause oscillations and performance degradation (b). 

Fig. 2. Latency distribution in four scenarios. 

the default autoscaler of Kubernetes<sup>2</sup> filters microservices for direct scaling based on a static threshold of computing resources. Yu et al. [15] defined the boundaries of elastic scaling by calculating the service power, which is the ratio between the 50th percentile response time (P50) and the 90th percentile response time(P90). Furthermore,Qiuetal. [4] introducedanSVM-based approach for extracting critical paths by analyzing the ratio of various tail latencies. Although these studies have narrowed the scope of autoscaling, they still take into account non-bottleneck microservices that may affect scaling strategies, especially when a large number of microservices in the application are abnormal at the same time. Consequently, there is an urgent need to pinpoint bottleneck microservices accurately before autoscaling. 

To balance resource consumption and performance, existing works have employed online optimization algorithms to find near-optimal autoscaling strategies. However, due to the vast range of possible strategies for autoscaling, these approaches requireasignificantnumberofattempts,whichwillbeproblematic for online applications. For example, Train Ticket<sup>3</sup> is the largest open-source microservice application, consisting of nearly 40 microservices. Assuming that each microservice can have up to 15 replicas, determining the optimal allocation strategy for this application is undoubtedly an NP-hard problem, as there are a maximumof 15<sup>40</sup> scalingalternatives.Additionally,theduration of the feedback loop in online optimization is too long to achieve model convergence. It is also essential to consider the potential risks of performance degradation caused by online optimization. Fig. 3 illustrates the impact of burst workloads on the replica fluctuationandlatencyfluctuationofMicroScaler [15],anonline autoscaling approach incorporating online Bayesian optimization to find the global minimizer of the total cost. The frequent online attempts to create replicas (Fig. 3(a)) caused by online optimization result in oscillations and performance degradation (Fig. 3(b)). As a result, we are inspired to design an offline optimization process fueled by feedback from a simulator. 

This paper presents PBScaler, a horizontal autoscaling framework designed to prevent performance degradation in microservice-based applications by identifying and addressing bottlenecks. Instead of optimizing resources for all abnormal microservices, as was done in previous work [11], [15], we propose 

> 2https://github.com/kubernetes/autoscaler 

> 3https://github.com/FudanSELab/train-ticket 

TopoRank, a random walk algorithm based on topological potential theory (TPT) to identify performance bottlenecks (PBs). By taking into account microservice dependencies and anomaly potential, TopoRank improves the accuracy and explainability of bottleneck localization. After identifying the PBs by TopoRank, PBScaler further employs a genetic algorithm to find nearly optimal strategies. To avoid application oscillation caused by excessive optimization, the process is conducted offline and is guidedbyanSLOviolationpredictor,whichsimulatestheonline application and provides feedback to the scaling strategies. The main contributions of the paper are summarized as follows: 

- r We propose PBScaler, a bottleneck-aware autoscaling framework designed to prevent performance degradation in a microservice-based application. By pinpointing bottlenecks, PBScaler can reduce unnecessary scaling and expedite the optimization process. 

- r We employ a genetic algorithm-based offline optimization process to optimize resource consumption while avoiding SLO violations. This process is guided by an SLO violation predictor and is designed to strike a balance between resource consumption and performance without disrupting online applications. 

- r We design and implement PBScaler in the Kubernetes system. To evaluate its effectiveness, we conduct extensive experiments with real-world and emulated workload injection on two widely-used microservice systems running in an online environment. Experimental results demonstrate that PBScaler outperforms several state-of-the-art elastic scaling methods. 

The rest of the paper is organized as follows. In Section II, we discuss the related work about bottleneck analysis and autoscaling for microservices. In Section III, we describe the overall system in detail. In Section IV, we present the evaluations and experimental results. Section V concludes our work and discusses the future research direction. 

## II. RELATED WORK 

With the advancement of cloud computing, numerous autoscaling methods for cloud resources, such as virtual machines or containers, have been proposed in academia and industry [16], [17], [18], [19]. However, autoscaling for microservices can be much more complicated due to the intricate dependencies between microservices. 

Authorized licensed use limited to: QUAID E AZAM UNIVERSITY. Downloaded on August 11,2026 at 11:27:28 UTC from IEEE Xplore.  Restrictions apply. 

606 

IEEE TRANSACTIONS ON SERVICES COMPUTING, VOL. 17, NO. 2, MARCH/APRIL 2024 

Performance bottleneck analysis, also known as root cause analysis, is a useful way for quickly locating the bottleneck responsible for the performance degradation of microservices, hence reducing the time and effort required for autoscaling. In this section, we will analyze the related work on bottleneck analysis and autoscaling for microservices. 

## _A. Bottleneck Analysis_ 

In recent years, numerous methods for bottleneck analysis in microservice scenarios have been developed, most of which rely on three types of data: logs, traces, and metrics. 1) Logs. Jia et al. [20] and Nandi et al. [21] first extracted templates and flows from normal-state logs, matched them with target logs, and filtered out abnormal logs. 2) Traces. Trace is an event trackingbased record that reproduces the request process between microservices. Several studies [22], [23], [24], [25] have been introduced to pinpoint the bottlenecks using traces. Yu et al. [22], [23] located bottlenecks by combining spectrum analysis and the PageRank algorithm on the dependency graph constructed by traces, while Mi et al. [24] presented an unsupervised machine learning prototype to learn the pattern of microservices and filter out abnormal microservices. However, using traces can be intrusive to the code and requires operators to have a deep understanding of the structure of the microservices. 3) Metrics. Some approaches [2], [26], [27] leverage graph random walk algorithms to simulate the propagation process of anomalies and then find bottlenecks by integrating statistical features of metrics and dependencies between microservices. Additionally, methods such as CauseInfer [14] and MicroCause [28] focused on building metrics causality graphs with causal inference, which typically involve hidden indirect relationships between metrics. 

Since the workflow code is rarely modified when monitoring metrics, collecting metrics for microservices is usually cheaper than using trace. Moreover, using metrics as the primary monitoring data can reduce the cost of integrating bottleneck analysis and autoscaling, as metrics are widely used in the latter scenario. Despite these approaches’ advantages, most have no preference in selecting the starting point for abnormal backtracking. In contrast, our approach begins random walks from microservices with greater anomaly potential, accelerating convergence speed and improving bottleneck localization accuracy. 

Using the control theory, SHOWAR [11] dynamically adjusts the microservice replicas to correct the error between monitoring metrics and thresholds. 4) Optimization-based approach. These methods [15], [34] make a large number of attempts to find the optimal strategy given the present resources and workloads. The key to these approaches is to reduce the decision-making scope to speed up the process. 5) RL-based Approach. Reinforcement learning (RL) has been widely used in resource management for microservices. MIRAS [35] adopts a model-based RL method for decision-making to avoid the high sampling complexity of the real environment. FIRM [4] leverages a support vector machine (SVM) to identify the critical path in microservices and a deep deterministic policy gradient (DDPG) algorithm to make hybrid scaling strategies for microservices along the path. RL-based methods require constant interactions with the environment during exploration and are incapable of adapting to the dynamic microservices architecture. 

In conclusion, while the aforementioned autoscaling techniques have their respective advantages, they pay little attention to performance bottlenecks. Consuming computer resources for non-bottleneck microservices will inevitably increase scaling costs and lengthen decision-making. Our method, on the other hand, focuses on locating performance bottlenecks. 

## III. SYSTEM DESIGN 

We present PBScaler, a PB-centric autoscaling controller, to locate PBs and optimize replicas for them. As shown in Fig. 4, PBScaler comprises three components: 1) _Metric Collector:_ To providereal-timeinsightsintotheapplications’status,wedesign ametriccollectorthatcapturesandintegratesmonitoringmetrics from Prometheus<sup>4</sup> at fixed intervals. 2) _Performance Bottleneck Analysis:_ With the assistance of the _metric collector_ , this component performs SLOviolation detection and redundancy checking to identify microservices with abnormal behavior. Next, the bottleneck localization process will be triggered to pinpoint the PBs in the abnormal microservices. 3) _Scaling Decision:_ This component aims to determine the optimal number of replicas for PBs using an evolutionary algorithm. Finally, PBScaler generates configuration files with optimized strategies and commits them to the kubernetes-client,<sup>5</sup> which regulates the replica count of microservices. 

## _A. Metric Collector_ 

## _B. Autoscaling for Microservices_ 

Existing autoscaling methods for microservices can be categorized into five groups. 1) Rule-based heuristic approach. KHPA, Libra [29], KHPA-A [30], and PEMA [31] manage the number of microservice replicas based on resource thresholds and specific rules. However, since different microservices have varying sensitivities to specific resources, expert knowledge is needed to support autoscaling for these various microservices. 2) Model-based approach. Microservices can be modeled to predict their status under particular configurations and workloads. Queuing theory [32], [33] and graph neural network (GNN) [12] are commonly used to build performance prediction models for microservices. 3) Control theory-based approach [11], [32]. 

The autoscaling controller relies on observability for microservice applications, such as system load, tail latency, and invocation relationships between microservices, to determine whether elastic scaling should be performed and how many resources should be allocated. While a trace-based monitor can reflect the dependencies and performance of microservices, it requires a deep understanding of the program and code injection [7]. Moreover, real-time analysis of massive tree-structural traces demands a considerable amount of processing time. Therefore, we design a metric-based monitor, the 

> 4https://prometheus.io 

> 5https://github.com/kubernetes-client/python 

Authorized licensed use limited to: QUAID E AZAM UNIVERSITY. Downloaded on August 11,2026 at 11:27:28 UTC from IEEE Xplore.  Restrictions apply. 

607 

XIE et al.: PBScaler: A BOTTLENECK-AWARE AUTOSCALING FRAMEWORK FOR MICROSERVICE-BASED APPLICATIONS 



<!-- Start of picture text -->
1 Deployed in Cloud 1 Timing Fluctuation i<br>' <Sten, Microservices P14: Metric Collector P2: Performance Bottlenec k Analysis<br>' 1) Metrics SLO Violation | "99°" (Bottleneck<br>1 1 | oe Detection _J True "|_Localization<br>'1 e Prometheus| | : False<br>!1 ||  SE Servi Service 1t : 'O Redundancy Bg<br>' ; ‘ Checking<br>1 N A 1 PBs<br>! iad yy ' P3: Scaling Decision<br>' Istio SLO Violati<br>!' {Service, I’ Control Plane| |1 eployment EN Strateg y iolation Predictor<br>1 Feedback| Workloads<br>1 {ener | i onfiguration |> DecisionMater | > Replicas => oh<br>'1 Gp eremetesController} vs1 KubernetesClient RandomandomForestForest<br>b------ cece eee<br><!-- End of picture text -->

Fig. 4. Framework of PBScaler. 

### TABLE I 

LABELS OF METRICS COLLECTED FROM THE MONITORING TOOLS 



<!-- Start of picture text -->
——SO—~—“—~‘a Tel<br>AT SOC~SOS<br>ube _ pod _i nfo<br>containe r_ cpu _u sage _s econds_ to tal<br>containe r_ memory _u sage _b ytes<br>containe r_ spe c_ cpu _q uota<br>Container Level containe_containe r_r_ fspe s_ us c_ agememo _b ytery_s li mit _b ytes<br>container _fs_Swe write _s econds _toin tal<br>container _fs_r ead _s econds_ to tal<br>containe r_ network_ re ceive _b ytes _to tal<br>containe r_ network _t ransmit_ by tes_ to tal<br>isti o_ reques t_ duratio n_ millisecond s_ bucket<br>Microservice Level _ isti o_ requests _t otal<br>isti o_ tc p_ received _b ytes _t otal<br><!-- End of picture text -->

_Metric Collector_ , based on non-intrusive service mesh technology to minimize disruptions to business flows. As shown in Table I, PBScaler uses Prometheus and kube-state-metrics to gather and categorize these metrics ( _M_ ), including tail latency, invocation relationships between microservices, resource consumption, and microservice workload. For example, _container_cpu_usage_seconds_total_ is a resource metric that records the Central Processing Unit (CPU) usage at the container level. _istio_requests_total_ records the TCP requests between microservices. Fig. 5 displays a visualization of the query results for _istio_requests_total_ , where each point represents the average request rate within the preceding minute for a specific invocation relationship (e.g., _frontend→currency_ as marked in Fig. 5). This information serves as a means to intuitively reflect the workload on microservices. Furthermore, _istio_requests_total_ also records all invocation relationships (listed in the legend of Fig. 5), which can be utilized to construct a microservice correlation graph _Gc_ similar to the one depicted in Fig. 6. Meanwhile, _istio_request_duration_milliseconds_bucket_ serves as a performance metric reflecting the tail latency of a microservice. Typically, we collect the P90 tail latency of each microservice to observe their performance. The monitoring interval of 



<!-- Start of picture text -->
6 =<br>@” ; | aim ped<br>© | pol ed<br>B. | \ peaewet<br>2 oped etatatoerane \<br>é& 6 ‘2029-12-28 171:20 banat<br>4 | © (to"currency’,from="rontend’) 7.00,<br>> | | pogo =<br>Peer sedeSos = 5500065550006555 =2 ses eeeeoorerresrss<br>ees att sot<br>Time ie<br>Oops, tom="tontond’) =i (tos"oureney’ om="rontone)<br>; ; ;<br>=@ {t0="recommend'”, from="frontend”) == {t0="product’, from="checkout")<br>=@ {t0="checkout", from="frontend"} =@— {to="product”, from="recommend”}<br>@> (10-"shipping*, from="trontend") oo Legend<br><!-- End of picture text -->

Fig. 5. Visualization of query results for metric _istio_requests_total_ . 



<!-- Start of picture text -->
(_Frontend_} <Z]\\~<br>DD reratmeoserice (GG sincmal mizosonice (7) estonect msosonce<br>—> Normal invocation —> Abnormal invocaton<br><!-- End of picture text -->

Fig. 6. Example of anomaly propagation in microservices. 

Prometheus is set to five seconds, and the collected metrics data is stored in a time-series database. 

_Service Mesh_ . A service mesh is an infrastructure that enables developers to add advanced features, such as observability and 

Authorized licensed use limited to: QUAID E AZAM UNIVERSITY. Downloaded on August 11,2026 at 11:27:28 UTC from IEEE Xplore.  Restrictions apply. 

608 

IEEE TRANSACTIONS ON SERVICES COMPUTING, VOL. 17, NO. 2, MARCH/APRIL 2024 

traffic management, to cloud applications without requiring additional code. One popular open-source service mesh implementation is Istio,<sup>6</sup> designed to seamlessly integrate with Kubernetes. When a pod starts up in Kubernetes, Istio launches an envoy proxy within the pod to intercept network traffic, enabling workload balancing and monitoring. 

## _B. Performance Bottleneck Analysis_ 

_Performance Bottleneck Analysis_ (PBA) is a process designed to discover performance degradation and resource waste in microservice applications to infer PBs of the current problem. As stated in Section I, by accurately locating these bottlenecks, PBA can enhance the performance of autoscaling and reduce excessive resource consumption. The PBA process in PBScaler is depicted in Algorithm 1. 

_1) SLO Violation Detection:_ To detect abnormalities in microservices, PBScaler uses service level objectives (SLOs) to compare with specific metrics. It is considered abnormal if a microservice has numerous SLO violations, i.e., performance degradation. As discussed in [14], [27], detecting SLO violations is a critical step in triggering bottleneck localization. The invocation relationships collected by the _Metric Collector_ can be leveraged to build a microservice correlation graph _Gc_ . PBScaler inspects the P90 tail latency of all invocation edges in _Gc_ every 15 seconds to timely detect performance degradation. If the tail latency of an invocation exceeds a predetermined threshold (such as the SLO), the invoked microservice of the invocation will be added to the set of abnormal microservices (S), and the bottleneck localization process will be activated. To account for occasional noise in the microservice latency, the threshold is set to SLO _×_ (1 +<sup>_<u>α</u>_</sup> 2<sup>), where</sup><sup>_α_is used to adjust the tolerance to</sup> noise. 

_2) Redundancy Checking:_ In the absence of performance anomalies, some microservices may be allocated more resources than required. However, identifying such cases can be difficult through metrics alone, potentially leading to wasting limited hardware resources. To avoid this, it is essential to identify which microservices have allocated excess resources. PBScaler uses the rate of workload change per second of microservices to determine whether resources are redundant. This strategy is more effective than relying only on resource consumption because different microservices may have varying sensitivity to heterogeneous resources. The main idea behind redundancy checking is to employ hypothesis testing to detect whether a microservice’s current workload **_w_**<sup>_i_</sup> **_c_**<sup>is significantly lower than</sup> its past workload (denoted as **_w_**<sup>_i_</sup> **_p_**<sup>).Thedegreeofsignificance</sup> is adjusted by the parameter _β_ , and the hypothesis test can be formulated as 



To perform the hypothesis test, we first fetch the current and historical workloads of the target microservices from the _Metric Collector_ . We then use a one-sided test to compute the p-value 

6https://istio.io 

|**Algori**|**thm 1:**Performance Bottleneck Analysis.|
|---|---|
|**Inp**|**ut:**SLO<br>|
||microservice correlation graph_Gc_<br>confidence level_cl_|
||tolerance to noise_α_|
||degree of significance_β_|
||impact factor_σ_|
|**Ou**|**tput:**ranking list**_rl_**|
|1: I|nitialize**_rl_**and abnormal microservice setS|
|2:**f**|**or**each_vi_in_Gc_.nodes**do**|
|3:|**for**each_ej,i_ in in-edges of_vi_**do**|
|4:|/* SLO violation detection */|
|5:|**if**_P_90(_ej,i_)_>_SLO_∗_(1 +_α/_2)**then**|
|6:|Store_vi_inS|
|7:|**end if**|
|8:|**end for**|
|9:**e**|**nd for**|
|10:|**if**Sis empty**then**|
|11:|/* Redundancy checking */|
|12:|**for**each_vi_in_Gc.nodes_**do**|
|13:|**_w_**<sup>_i_</sup><br><sup>,</sup><sup>**_w_**</sup><sup>_i_</sup><br>**_c_** <sup>_←_Get workloads for microservice</sup><sup>_i_</sup>|
||**_p_**<br> <br>from the Metric Collector<br><br>|
|14:|_t_,_P ←ttest_(**_w_**<sup>_i_</sup><br>**_c_**<sup>,</sup><sup>**_w_**</sup><sup>_i_</sup><br>**_p_** <sup>_∗β_)</sup>|
|15:|**if**_t <_0and_P < cl_**then**|
|16:|Store_vi_in**_rl_**|
|17:|**end if**|
|18:|Return**_rl_**|
|19:|**end for**|
|20:|**end if**|
|21:|**_rl_**=_TopoRank_(S)|
|22:|Return**_rl_**|



_P_ . If _P_ does not exceed the confidence level _cl_ (which is set to 0.05 by default), we reject the null hypothesis _H_ 0 and consider the microservice _i_ to have redundant resources. 

_3) Bottleneck Localization:_ Because of the complex interactions in a microservice application [36], not every abnormal microservice needs to be scaled up. For example, Fig. 6 illustrates how the performance degradation of a bottleneck microservice (e.g., _Product_ ) can propagate to its upstream microservices (e.g., _Recommend_ , _Frontend_ ,and _Checkout_ )alongtheinvocation chains, even if the upstream microservices are not overloaded. Therefore, only the bottleneck microservice must be scaled up while the other abnormal microservices are merely implicated. To pinpoint the bottleneck microservice, we introduce the concept of anomaly potential, which aggregates the anomaly impact of all microservices in a given position. Generally, a performancebottlenecktendstoexhibitahighanomalypotential in a microservice application, as it is frequently surrounded by abnormal neighbors that are influenced by the bottleneck. Based on the anomaly potential, we model the backtracking of anomalies in microservice applications as a random walk on a directed graph. Therefore, We design a novel bottleneck localization algorithm, TopoRank, which introduces the topological potential theory (TPT) in random walks to calculate the scores 

Authorized licensed use limited to: QUAID E AZAM UNIVERSITY. Downloaded on August 11,2026 at 11:27:28 UTC from IEEE Xplore.  Restrictions apply. 

609 

XIE et al.: PBScaler: A BOTTLENECK-AWARE AUTOSCALING FRAMEWORK FOR MICROSERVICE-BASED APPLICATIONS 







Fig. 7. Illustration of the computation process for the TopoRank algorithm. 

microservices and their invocation relationships in _Gc_ . We then calculate the anomaly potential _ϕ_ for microservice _vi_ in the abnormal subgraph _Ga_ using the TPT 

|**Algo**<br>|**rithm 2:**TopoRank.<br>|p _c_.<br>calculate the anomaly potential _ϕ_ for microservice _vi_ in the|
|---|---|---|
|**Inp**|**ut:**abnormal subgraph_Ga_<br>impact factor_σ_|<br>abnormal subgraph_Ga_ using the TPT|
||preference vector**_u_**||
||transition matrix**_P_**<br>collected metrics_M_|_ϕ_=_ai_+<br>_N_<br>�<br>_aje_<sup>_−_(</sup><sup>_hji/σ_)2</sup>_,_<br>(2)|
|**Out**|**put:**ranking list**_rl_**|_j_=1|
|1:|Initialize**_rl_**and preference vector**_u_**||
|2:<br>|**for**each_vi_in_Ga_.nodes**do**<br>|where _N_ is the number of upstream microservices of _vi_ and|
|3:<br>|_ai ←_Anomaly degree of_vi_|_aj_ represents the anomaly degree of _vj_. PBScaler defines the|
|4:<br>|_ϕ ←ai_<br>|anomaly degree as the number of SLO violations for a microser-|
|5:<br>|**for**each_vj_ in upstream microservices of_vi_**do**<br>|vice in a time window._hji_denotes the minimum number of hops|
|6:<br>|_hji ←_Minimum number of hops from_vj_ to_vi_<br>|required from_vj_ to_vi_. We use the impact factor_σ_(1 by default)|
|7:|_aj ←_Anomaly degree of_vj_<br><sup>2</sup>|to control the influence range of a microservice.|
|8:|_ϕ ←ϕ_+_aje_<sup>_−_(</sup><sup>_hji/σ_)</sup>|Fig.7illustrates the computation process of the TopoRank al-|
|9:|**end for**|gorithm. PBScaler checks the number of SLO violations for each|
|10:|**_u_**_i ←ϕ_|microservice in Step 1 to initialize the anomaly degree. It can be|
|11:|_Lt ←_An array of tail latency for_vi_|observed that the microservices A and C, positioned upstream|
|12:|**for**each_vj_ in out-edges of_vi_**do**|of the performance bottleneck (i.e., microservice E), have the|
|13:|_Lm ←_An array of metric_m_for_vj_<br>|highest anomaly degrees, with values of 12 and 8, respectively.|
|14:|**_P_**_i,j ←_max_m∈M_(_corr_(_Lt, Lm_))|This is because anomalies in downstream microservices propa-|
|15:|**end for**|gate and accumulate in upstream microservices. Consequently,|
|16:|**end for**|it is inadvisable to locate PBs solely based on the anomaly|
|17:|**_rl_**_←pageRank_(**_P_**,**_u_**)|degree. In Step 2, PBScaler calculates the anomaly potential|
|18:|Return**_rl_**|for each microservice according to(2)Taking the microservice|



Fig. 7 illustrates the computation process of the TopoRank algorithm. PBScaler checks the number of SLO violations for each microservice in Step 1 to initialize the anomaly degree. It can be observed that the microservices A and C, positioned upstream of the performance bottleneck (i.e., microservice E), have the highest anomaly degrees, with values of 12 and 8, respectively. This is because anomalies in downstream microservices propagate and accumulate in upstream microservices. Consequently, it is inadvisable to locate PBs solely based on the anomaly degree. In Step 2, PBScaler calculates the anomaly potential for each microservice according to (2). Taking the microservice C as an example, its anomaly potential _ϕ_ can be calculated as: 8 + 12 _× e_<sup>_−_(1</sup><sup>_/_1)2</sup> = 12 _._ 41. 

However, microservices with high anomaly potential values are not necessarily PBs, since anomalies are usually propagated along the microservice correlation graph. In Step 2 of Fig. 7, the upstream microservice A exhibits the highest anomaly potential due to the propagation of downstream anomalies. Hence, relying solely on the TPT is insufficient for diagnosing PBs. To address this issue, PBScaler incorporates the Personalized PageRank algorithm [39] toreversetheanomalypropagationontheabnormal subgraph _Ga_ and locate PBs. Let **_P_** be the transition matrix of _Ga_ and **_P_** _i,j_ be the probability of anomaly tracking from _vi_ to its downstream node _vj_ . Given _vi_ with out-degree _d_ , the standard Personalized PageRank algorithm sets **_P_** _i,j_ as 

for all abnormal microservices and finally outputs a ranking list ( **_rl_** ). The microservices with the highest scores in the **_rl_** can be recognized as the PBs. 

TPT, which originated from the concept of “field” in physics, has been widely used in various works [37], [38] to measure the mutual influence between nodes in complex networks. Since the microservice correlation graph can also be viewed as a complex network, we use TPT to evaluate the anomaly potential of microservices. Specifically, we have observed that in a microservice correlation graph _Gc_ , the microservices closer to the PBs, i.e., those with fewer hops, are more likely to be abnormal, as they often have frequent direct or indirect invocations with the PBs. Based on this observation, we evaluate the anomaly potential of microservices using the TPT. To do this, we first extract the abnormal subgraph _Ga_ by identifying the abnormal 



Authorized licensed use limited to: QUAID E AZAM UNIVERSITY. Downloaded on August 11,2026 at 11:27:28 UTC from IEEE Xplore.  Restrictions apply. 

610 

IEEE TRANSACTIONS ON SERVICES COMPUTING, VOL. 17, NO. 2, MARCH/APRIL 2024 

which means that the algorithm is not biased toward any downstream microservice. However, this definition fails to consider the association between the downstream microservices and the anomaly of the current microservice. Consequently, PBScaler adapts the calculation by giving more attention to the downstream microservices whose metrics are more relevant to upstream response time. For each microservice _vi_ , PBScaler logs a tail latency array ( _Lt_ ). For any metric _m_ (e.g., _container_  spec_  cpu_  quota_ ) of collected metrics _M_ in Table I, PBScaler records a metric array _Lm_ for _vi_ within a specified time window. PBScaler defines that **_P_** _i,j_ depends on the maximum value of the Pearson correlation coefficient between _Lt_ and _Lm_ 



The Personalized PageRank algorithm determines the popularity of each node by randomly walking on the directed graph. However, some nodes may never point to others, causing the scores of all nodes to tend toward zero after many iterations of the random walk. To avoid falling into this “trap,” a damping factor _δ_ is applied, which allows the algorithm to jump out from these nodes according to a predefined rule. Typically _δ_ is set to 0.15. The Personalized PageRank is represented as follows: 



where **_v_** represents the probability that each microservice node is diagnosed as a PB. The preference vector **_u_** serves as the personalized rule to guide the algorithm to leap from the trap. The value of **_u_** is determined by the anomaly potential _ϕ_ of each node. The nodes with greater anomaly potential are preferred as starting points for the algorithm. The equation of the _k_ th iteration can be represented as 



After multiple rounds of iterations, **_v_** gradually converges. PBScaler then sorts the final results and produces the ranking list **_rl_** . In Step 4 of Fig. 7, the TopoRank algorithm undergoes multiple rounds of random walks. During this process, it reduces the suspicion levels on microservices A and C, assigning them low scores of 0.11 and 0.17, respectively. Conversely, microservice E is identified with the highest suspicion score of 0.37. The top- _k_ microservices (E and D) with the highest ranking list scores can be recognized as PBs. The whole process of TopoRank is depicted as Algorithm 2. 

## _C. Scaling Decision_ 

Given the PBs identified by the _Performance Bottleneck Analysis_ , the replicas for the PBs will be scaled to minimize the application’s resource consumption while ensuring the end-toend latency of microservices meets the SLO. Although abundant replicas can alleviate the performance degradation problem, they also consume a significant amount of resources. Consequently, it is essential to maintain a balance between performance guarantees and resource consumption. The process of _Scaling Decision_ will be modeled as a constrained optimization problem to achieve this balance. 

_1) Constrained Optimization Model:_ The autoscaling optimization in our scenario seeks to identify an allocation schema that allocates a variable number of replicas for each PB. Given _n_ PBs that require scaling, we define a strategy as a set X = _{x_ 1 _, x_ 2 _, . . . , xn}_ , where _xi_ denotes the number of replicas allocated for PB _i_ . Before the optimization, the initial number of replicas for PBs can be expressed as C = _{c_ 1 _, c_ 2 _, . . . , cn}_ . It should be noted that the replicas constraint in PBScaler should be defined separately for scaled-down and scaled-up processes. During the scaled-up process, we limit the number of replicas for PBs as follows: 



where _c_<sup>max</sup> represents the maximum number of replicas that a microservice can scale to, given limited server resources. The constraint of the number of replicas during the scaled-down process can be expressed as 



In (8), _γ_ (with the default value of two) denotes the maximum number of replicas reductions. This limit is reasonable since reducing the number of microservice replicas drastically can cause a short latency peak, as observed in experiments. 

The goal of the _Scaling Descision_ is to minimize the application’s resource consumption while maintaining its performance. Application performance is usually expressed by SLO violations that users are more concerned about. Therefore, the application performance reward can be detailed as 



During the optimization process, the application’s resource consumption, such as CPU and memory usage, is unpredictable. To conservatively estimate resource consumption, we consider the ratio of PB replicas to the maximum number of allocatable replicas, rather than calculating the cost of CPU and memory. We calculate the resource reward as 



Our objective is to guarantee performance while minimizing resource consumption. We leverage a weighted linear combination (WLC) method to balance the two objectives. The final optimization objective is defined as 



where _λ ∈_ [0 _,_ 1]. We set _λ_ as a parameter to balance the application performance and resource consumption. 

_2) SLO Violation Predictor:_ To calculate the performance reward _R_ 1, evaluating whether a strategy will cause the SLO violation in online applications is necessary. A simple way is to execute candidate strategies directly in online applications and wait for feedback from the monitoring system. However, oscillations caused by frequent scaling in online applications 

Authorized licensed use limited to: QUAID E AZAM UNIVERSITY. Downloaded on August 11,2026 at 11:27:28 UTC from IEEE Xplore.  Restrictions apply. 

611 

XIE et al.: PBScaler: A BOTTLENECK-AWARE AUTOSCALING FRAMEWORK FOR MICROSERVICE-BASED APPLICATIONS 

will be inevitable. An alternative method is to train an evaluation model with historical metric data, which can simulate the feedback from online applications. Without interacting with online applications, this model predicts application performance based on the current application state. 

We use a vector **_r_** to denote the number of replicas for each microservice after executing the scaling strategy X. **_w_** is a vector that denotes the current workload for each microservice. Because of the low time cost of bottleneck-aware optimization, it is reasonable to hypothesize that **_w_** will not change significantly during this period (see Section IV-B). Given the application state represented by workload **_w_** and replicas **_r_** of all microservices, an SLO violation predictor can be designed as 



where _ψ_ is a binary classification model. Details of selecting an appropriate classification model will be discussed in Section IV-C. The historical metric data used for training can be generated using either a classical scaling method (the Kubernetes autoscaler by default) or a stochastic method. We deployed an open-source microservice system on three nodes (with a total of 44 CPU cores and 220 GB of RAM) and performed elastic scaling. Prometheus gathered each microservice’s workload and P90 tail latency at regular time intervals. By comparing the tail latency of the front-end microservice with the SLO, monitoring data for each time interval can be easily labeled. 

_3) Autoscaling Optimization:_ As mentioned in Section III-C1, the tradeoff between performance and resource consumption can be modeled as a constrained optimization problem. To find a near-optimal strategy, PBScaler employs a genetic algorithm (GA) to generate and optimize scaling strategies that reduce resource consumption while meeting SLO requirements. By emulating natural selection in evolution, the GA improves the superior offspring while eliminating the inferior ones. Initially, the GA performs a random search to initialize a population of chromosomes in the strategy space, with each chromosome indicating a potential strategy for the optimization problem. Next, in each iteration, elite chromosomes with high fitness, called elites, will be selected for crossover or mutation to produce the next generation. 

The autoscaling optimization in our scenario seeks to identify a scaling strategy that allocates a variable number of replicas for each PB. The process of autoscaling optimization is illustrated in Fig. 8. In the beginning, PBScaler obtains each microservice’s current number of replicas **_r_** and workload **_w_** . After the _Performance Bottleneck Analysis_ , PBScaler identifies the PBs from **_r_** and filters out them to get **_r_**<sup>**_′_**</sup> . Then, the population of strategies for PBs is generated by the Decision Maker. Since the number of microservices to be scaled influences the speed and effect of the optimization algorithm (Section IV-C), PBScaler assumes that only PBs need to be elastically scaled. In other words, the number of replicas in **_r_**<sup>**_′_**</sup> will remain unchanged. The SLO violation predictor is responsible for evaluating the generated strategies. It should be noted that the strategy is merged with **_r_**<sup>**_′_**</sup> and input to the SLO violation predictor together with **_w_** . With 

- **Algorithm 3:** GA-Based Autoscaling Optimization. **Input:** the number of iterations _I_ the size of population _Np_ the size of elites _Nz_ the probability of crossover _pc_ the probability of mutation _pm_ 

- **Output:** the best scaling strategy _s_ 1: _P_ 0 _←_ Randomly create a population with _Np_ strategies 2: Evaluate the fitness of each strategy in _P_ 0 3: _Z_ 0 _←_ Get _Nz_ elites from _P_ 0 4: _s ←_ Get the best strategy in _Z_ 0 5: _i ←_ 0 6: **while** _i < I_ **do** 7: _Fi ←_ Set parents with ( _Np − Nz_ ) strategies from _Pi_ 8: _Oi ←_ Generate offspring by recombining _Fi_ with a probability of _pc_ 

- 9: _Oi ←_ Generate offspring by mutating _Oi_ with a probability of _pm_ 

- 10: _Pi_ +1 _←_ ( _Zi ∪ Oi_ ) 11: Evaluate the fitness of each strategy in _Pi_ +1 12: _Zi_ +1 _←_ Get _Nz_ elites from _Pi_ +1 13: _t ←_ Get the best strategy in _Zi_ +1 14: _fs, ft ←_ Evaluate the fitness of _s_ and _t_ 15: **if** _fs < ft_ **then** 16: _s ← t_ 17: **end if** 18: _i ← i_ + 1 <u>19:</u> **<u>end while</u>** 



Fig. 8. Illustration of the autoscaling optimization process. 

the help of GA, the superior strategy X _best_ is selected and then merged with **_r_**<sup>**_′_**</sup> to generate the final decision. 

In the optimization phase, the Decision Maker generates and improves the scaling strategy for PB using the GA, as described in Algorithm 3. After randomly generating a population within the strategy scope of each PB (Line 1), the Decision Maker estimates the fitness of each strategy based on (11) and stores the elites (Lines 2-3). In each iteration, the GA uses a tournamentbased selection operator to pick out outstanding parents _Fi_ 

Authorized licensed use limited to: QUAID E AZAM UNIVERSITY. Downloaded on August 11,2026 at 11:27:28 UTC from IEEE Xplore.  Restrictions apply. 

612 

IEEE TRANSACTIONS ON SERVICES COMPUTING, VOL. 17, NO. 2, MARCH/APRIL 2024 

(Line 7). New offspring _Oi_ are generated through recombination and mutation (Lines 8-9) using a two-point crossover operator and a binary-chromosome mutation operator. By simulating natural selection, new offspring _Oi_ and elites _Zi_ with higher fitness constitute a new population _Pi_ +1 that enters the next iteration (Line 10). At each iteration, the current best strategy _t_ undergoesafitnesscomparisonwiththehistoricallybeststrategy _s_ (Lines 13–17). The best strategy _s_ after _I_ iterations will be considered as the value of final X _best_ . 

## IV. EVALUATIONS 

In this section, we present the details of experimental scenarios for autoscaling, including a comparison of PBScaler with several state-of-the-art autoscaling algorithms from academia and industry. 

## _A. Experimental Setup_ 

_1) Microservice Platform:_ Experiments were carried out in our private cloud cluster, consisting of three physical computers (one master node and two worker nodes) with a total of 44 Intel 2.40 GHz CPU cores and 220 GB of RAM. To evaluate autoscaling, we selected two open-source microservice applications as benchmarks: a) Online Boutique,<sup>7</sup> a Web-based E- commerce demo application developed by Google. The system implements basic functions such as browsing products, adding items to shopping carts, and payment processing through the collaboration of ten stateless microservices and a Redis cache. b) Train Ticket:<sup>8</sup> a large-scale, open-source microservice system developed by Fudan University. With more than 40 microservices and the usage of MongoDB and MySQL for data storage, Train Ticket can satisfy a variety of workflows, such as online ticket browsing, booking, and purchasing. Because of cluster resource constraints, we limited each microservice to no more than eight replicas. The source code is available on Github.<sup>9</sup> 

_2) Workload:_ We evaluated the effectiveness of PBScaler under various traffic scenarios, using a real-world Wikipedia workload from the Wiki-Pageviews [40] on March 16, 2015, and five emulated workloads (EW1 _∼_ EW5), inspired by the experiments conducted by Abdullah et al. in [41]. We compressed the real-world workload to one hour and scaled it to an appropriate level for our cluster. The five emulated workloads exhibited various patterns, such as single peak, multiple peaks, rising, and dropping, and were limited to a duration of twenty minutes. Fig. 9 depicts the fluctuation of these workloads. 

_3) Baseline Methods:_ We compare PBScaler with several state-of-the-art microservice autoscaling methods from academia and industry, which perform dynamic horizontal scaling of microservices from the perspectives of static thresholds, control theory, and black-box optimization. 

- r _Kubernetes Horizontal Pod Autoscaling (KHPA):_ It is the default horizontal scaling scheme of Kubernetes. By customizing a threshold _T_ for a certain resource _R_ (CPU usage 

> 7https://github.com/GoogleCloudPlatform/microservices-demo 

> 8https://github.com/FudanSELab/train-ticket 

> 9https://github.com/WHU-AISE/PBScaler 



Fig. 9. Fluctuation of one-hour real-world workloads (Wiki) and twenty minutes emulated workloads (EW). 

- as the default) and aggregating the resource usage _Ui_<sup>_R_from</sup> all replicas of a microservice, KHPA defines the target number of replicas as _n_ = _⌈_<sup>�</sup> _i∈ActiveP ods_<sup>_U R_</sup> _i_<sup>_/ T⌉_.</sup> 

- r _MicroScaler [15]:_ It is an autoscaling tool that uses a black-box optimization algorithm to determine the optimal number of replicas for a microservice. MicroScaler calculates the microservice’s P90/P50 for classification and then performs four iterations of Bayesian Optimization to make a scaling decision. 

- r _SHOWAR [11]:_ It is a hybrid autoscaling technology. We reproduced the horizontal scaling part in SHOWAR, which uses the PID control theory to gradually bring the observed metric close to the user-specified threshold. In our implementation, we replaced the run queue latency with the more common P90 latency since the former requires an additional eBPF tool. 

_4) Experimental Parameters and Evaluation Criteria:_ In our experiments, we fixed the collection interval of Prometheus to five seconds. With the increase in experiment time and workloads, the data volume required by stateful microservices like MongoDB will also grow. Eventually, the data volume will exceed the available memory, necessitating the use of disk storage. Thistransitioncancauseperformancedegradationthatcannotbe remedied through autoscaling. Hence, we limit workload testing to stateless traces. The SLO values for the Online Boutique and the Train Ticket were set to 500 ms and 200 ms, respectively. In the SLO violation detection and redundancy checking module, PBScaler first sets the action boundary _α_ to 0.2 to reduce noise interference, as done in SHOWAR. Then, we empirically set the degree of significance _β_ to 0.9 to control the workload level that triggers scaling. For bottleneck localization, the impact factor _σ_ of the topological potential is set to 1, and the top- _k_ ( _k_ = 2) microservices with the highest score in **_rl_** will be considered PBs. 

We choose the SLO violation rate, resource consumption, and response time to evaluate the performance of autoscaling methods. An autoscaling approach is considered more effective if it can reduce response time, SLO violation rate, and resource consumption. We define the SLO violation rate as the percentage of the end-to-end P90 tail latency that exceeds the 

Authorized licensed use limited to: QUAID E AZAM UNIVERSITY. Downloaded on August 11,2026 at 11:27:28 UTC from IEEE Xplore.  Restrictions apply. 

613 

XIE et al.: PBScaler: A BOTTLENECK-AWARE AUTOSCALING FRAMEWORK FOR MICROSERVICE-BASED APPLICATIONS 

TABLE II 

PERFORMANCE OF STATE-OF-THE-ART METHODS AND PBSCALER UNDER REAL AND EMULATED WORKLOADS 



Fig. 10. Latency distribution for different methods under real and emulated workloads. 

SLO. Resource consumption is calculated following the method presented in [42], where the CPU price is 0.00003334$(vCPU/s) and the memory price is 0.00001389$(G/s). The total resource consumption is obtained by summing the cost of memory and CPU. 

## _B. Performance Evaluation_ 

Table II compares the SLO violation rates and resource costs for the four autoscaling methods in two microservice applications with different workloads. The None method is used as a reference and performs no autoscaling operation. Its results are presented in grey and are excluded from the comparison. 

In general, PBScaler outperforms the competing approaches in reducing SLO violations and minimizing resource overhead under six workloads in both microservices systems. In particular, the SLO violation rate of PBScaler in Train Ticket is, on average, 4.96% lower than that of the baseline methods, while the resource cost is reduced by an average of 0.24$. These results show that PBScaler can perform elastic scaling for bottleneck 

microservices in large-scale microservice systems quickly and precisely,therebyreducingSLOviolationsandsavingresources. Regarding the six workloads in Online Boutique, PBScaler also achieves the lowest SLO violations in four of them and minimizes resource consumption in three emulated workloads. 

Fig. 10 depicts the box plots of latency distribution for different methods under six workloads, exploring the impact of each method on the performance of the microservice system. It can be seen that the majority of the autoscaling methods can keep the median of the latency distribution below the red dotted line (SLO). However, only PBScaler goes a step further to reduce the third quartile significantly below the SLO for all workloads. 

To evaluate the time cost of using PBScaler for elastic scaling, the average time required by each module in PBScaler is collected and counted. As reported in Table III, the total time cost of all PBScaler modules in Online Boutique is less than one monitoring interval (i.e., 5 s), while the same metric for Train Ticket is less than two monitoring intervals. Thanks to the PBA that narrows the decision-making scope, the time cost of the Decision Maker does not increase much (no more than 6.6%) 

Authorized licensed use limited to: QUAID E AZAM UNIVERSITY. Downloaded on August 11,2026 at 11:27:28 UTC from IEEE Xplore.  Restrictions apply. 

614 

IEEE TRANSACTIONS ON SERVICES COMPUTING, VOL. 17, NO. 2, MARCH/APRIL 2024 

TABLE III 

TIME COST OF FOUR MODULES IN PBSCALER 



Fig. 11. Performance comparison of bottleneck localization on Online Boutique (OB) and Train Ticket (TT). 

when the application is switched from Online Boutique to Train Ticket, despite the increased number of microservices. However, we recognize the limitation that the time consumption of PBA quickly rises as the microservice scale grows, which will be our future work. 

## _C. Effectiveness Analysis of Components_ 

_1) Performance Comparision of Bottleneck Localization:_ To evaluate whether the TopoRank algorithm can effectively locate PBs caused by burst workloads, we injected exceptions, such as CPU overload, memory overflow, and network congestion, into Online Boutique and Train Ticket through Chaos Mesh. These exceptions are typically caused by high-workload conditions. The TopoRank algorithm was used to analyze the metrics and identify performance bottlenecks for these exceptions. The localization results were then compared to MicroRCA [27], a baseline method for microservice root cause analysis. _AC_ @ _k_ measures the accuracy of the real PBs in the top- _k_ results, and _Avg_ @ _k_ is the average accuracy at the top- _k_ results. These metrics can be calculated as follows. 





where _A_ represents the set of exceptions, and _RT_ @ _k_ refers to the top- _k_ microservices in the ranking list. Fig. 11 presents the _AC_ @1 and _Avg_ @5 values of TopoRank and MicroRCA across different microservice applications. The results indicate that TopoRank performs better than MicroRCA in both metrics. This is primarily due to the fact that TopoRank takes into account both the anomaly potential and microservice dependencies when performing Personalized PageRank. 



Fig. 12. Iterative process of the genetic algorithm under Online Boutique and Train Ticket. 

The primary purpose of bottleneck location is to narrow down the strategy space and expedite the discovery of the optimal strategy. We perform GA iterations on both PBs and all microservices to demonstrate the influence of bottleneck localization on optimization. Fig. 12 depicts the iterative process under the microservice systems and demonstrates that as the population increases, the PB-aware strategy significantly outperforms the approach that scales for all microservices in terms of fitness. The PB-aware strategy can obtain superior fitness in less than five iterations. In contrast, the all-microservices-involved method requires larger populations and more iterations to achieve the same level of fitness. This is attributed to the fact that the PB-aware strategy aids the genetic algorithm in reducing the optimization range precisely and accelerating the acquisition of superior solutions. 

_2) Effectiveness of the SLO Violation Predictor:_ The objective of the SLO violation predictor is to forecast the result of the optimization strategy directly rather than waiting for feedback from the online application. We determine whether performance issues will occur based on the number of replicas and workloads of each microservice. Selecting a suitable binary classification model for the prediction task is critical. With a data collection intervaloffiveseconds,wecollectedtwodatasets,includinga3.1k historical samplingdataset for TrainTicket anda1.5kdataset for Online Boutique, in our cluster. For training and testing on these two datasets, we adopt four classical machine learning (ML) methods, including Support Vector Machine (SVM), Random Forest, Multilayer Perceptron (MLP), and Decision Tree. We employ typical binary classification indicators to assess the performance of experimental methods: _Precision_ = _T PT P_ + _F P_ and _Recall_ = _T PT P_ + _F N_<sup>,where</sup><sup>_TP_representsthenumberof</sup> identified SLO violations; _FN_ and _FP_ represent the number of unrecognized SLO violations and the number of false-positive SLO violations, respectively. Table IV shows the performance comparison of the four methods for SLO violation prediction. 

Authorized licensed use limited to: QUAID E AZAM UNIVERSITY. Downloaded on August 11,2026 at 11:27:28 UTC from IEEE Xplore.  Restrictions apply. 

615 

XIE et al.: PBScaler: A BOTTLENECK-AWARE AUTOSCALING FRAMEWORK FOR MICROSERVICE-BASED APPLICATIONS 

TABLE IV 

PRECISION AND RECALL OF FOUR ML METHODS FOR SLO VIOLATION PREDICTION 



Fig. 13. Replica and latency fluctuation of PBScaler and MicroScaler under the Wiki workload. According to the effects of the two datasets, we finally choose Random Forest as the primary algorithm for the SLO violation predictor. 

To demonstrate that the SLO violation predictor can substitute the feedback from the real environment, we compare PBScaler, which employs the SLO violation predictor, with MicroScaler, which collects feedback from the online system. We injected burst workloads into the Online Boutique and made only one microservice abnormal to eliminate the difference in bottleneck localization between the two methods. As shown in Fig. 13, with the guidance of the predictor, the number and the frequency of decision-making attempts made by PBScaler are much lower than those of MicroScaler. Reducing online attempts in a cluster will evidently reduce the risk of oscillations. 

## V. CONCLUSION 

This paper presents PBScaler, a bottleneck-aware autoscaling framework designed to prevent performance degeneration in microservice-based applications. PBScaler collects real-time performance metrics of applications using the service mesh technology and dynamically builds a correlation graph among microservices. To handle abnormal microservices caused by external dynamic workloads and intricate invocations among microservices, PBScaler employs TopoRank, a random walk algorithm based on the topological potential theory, to identify bottleneck microservices. Furthermore, PBScaler performs an offline evolutionary algorithm to optimize scaling strategies guided by an SLO violation predictor. Experimental results indicate that PBScaler can minimize resource consumption while achieving lower SLO violations. 

In the future, we plan to improve our work from the following two aspects. First, we will explore the potential of using bottleneck awareness in finer-grained resource (e.g., CPU and memory) management. Second, we will explore how to circumvent the interference of stateful microservices in autoscaling since the performance degradation from stateful microservices 

may disrupt the autoscaling controller. Third, we will improve the efficiency of performance bottleneck analysis for large-scale microservice systems. 

## REFERENCES 

- [1] M. Ma, J. Xu, Y. Wang, P. Chen, Z. Zhang, and P. Wang, “AutoMAP: Diagnose your microservice-based web applications automatically,” in _Proc. Web Conf._ , 2020, pp. 246–258. 

- [2] M. Kim, R. Sumbaly, and S. Shah, “Root cause detection in a serviceoriented architecture,” _ACM SIGMETRICS Perform. Eval. Rev._ , vol. 41, no. 1, pp. 93–104, 2013. 

- [3] H. Shan et al., “ _ϵ_ -diagnosis: Unsupervised and real-time diagnosis of small-window long-tail latency in large-scale microservice platforms,” in _Proc. World Wide Web Conf._ , 2019, pp. 3215–3222. 

- [4] H. Qiu, S. S. Banerjee, S. Jha, Z. T. Kalbarczyk, and R. K. Iyer, “FIRM: An intelligent fine-grained resource management framework for SLOOriented microservices,” in _Proc. 14th USENIX Symp. Operating Syst. Des. Implementation_ , 2020, pp. 805–825. 

- [5] B. Liu, R. Buyya, and A. Nadjaran Toosi, “A fuzzy-based auto-scaler for web applications in cloud computing environments,” in _Proc. Int. Conf. Service-Oriented Comput._ , Springer, 2018, pp. 797–811. 

- [6] S. Zhang et al., “Robust system instance clustering for large-scale web services,” in _Proc. ACM Web Conf._ , 2022, pp. 1785–1796. 

- [7] J. Lin, P. Chen, and Z. Zheng, “Microscope: Pinpoint performance issues with causal graphs in micro-service environments,” in _Proc. Int. Conf. Service-Oriented Comput._ , Springer, 2018, pp. 3–20. 

- [8] D. Liu et al., “MicroHECL: High-efficient root cause localization in largescale microservice systems,” in _Proc. IEEE/ACM 43rd Int. Conf. Softw. Eng.: Softw. Eng. Pract._ , 2021, pp. 338–347. 

- [9] B. Choi, J. Park, C. Lee, and D. Han, “pHPA: A proactive autoscaling framework for microservice chain,” in _Proc. 5th Asia-Pacific Workshop Netw._ , 2021, pp. 65–71. 

- [10] A. Kwan, J. Wong, H.-A. Jacobsen, and V. Muthusamy, “HyScale: Hybrid and network scaling of dockerized microservices in cloud data centres,” in _Proc. IEEE 39th Int. Conf. Distrib. Comput. Syst._ , 2019, pp. 80–90. 

- [11] A. F. Baarzi and G. Kesidis, “SHOWAR: Right-sizing and efficient scheduling of microservices,” in _Proc. ACM Symp. Cloud Comput._ , 2021, pp. 427–441. 

- [12] J.Park,B.Choi,C.Lee,andD.Han,“GRAF:Agraphneuralnetworkbased proactive resource allocation framework for SLO-oriented microservices,” in _Proc.17thInt.Conf.Emerg.Netw.EXperimentsTechnol._ ,2021,pp.154– 167. 

- [13] Y. Al-Dhuraibi, F. Paraiso, N. Djarallah, and P. Merle, “Elasticity in cloud computing: State of the art and research challenges,” _IEEE Trans. Services Comput._ , vol. 11, no. 2, pp. 430–447, Mar./Apr. 2018. 

- [14] P. Chen, Y. Qi, and D. Hou, “CauseInfer: Automated end-to-end performance diagnosis with hierarchical causality graph in cloud environment,” _IEEE Trans. Serv. Comput._ , vol. 12, no. 2, pp. 214–230, Mar./Apr. 2019. 

- [15] G. Yu, P. Chen, and Z. Zheng, “Microscaler: Automatic scaling for microservices with an online learning approach,” in _Proc. IEEE Int. Conf. Web Serv._ , 2019, pp. 68–75. 

- [16] U. Sharma, P. Shenoy, S. Sahu, and A. Shaikh, “A cost-aware elasticity provisioningsystemforthecloud,”in _Proc.31stInt.Conf.Distrib.Comput. Syst._ , 2011, pp. 559–570. 

- [17] R. da Rosa Righi, V. F. Rodrigues, C. A. Da Costa, G. Galante, L. C. E. De Bona, and T. Ferreto, “AutoElastic: Automatic resource elasticity for high performance applications in the cloud,” _IEEE Trans. Cloud Comput._ , vol. 4, no. 1, pp. 6–19, First Quarter 2016. 

- [18] N. Roy, A. Dubey, and A. Gokhale, “Efficient autoscaling in the cloud using predictive models for workload forecasting,” in _Proc. IEEE 4th Int. Conf. Cloud Comput._ , 2011, pp. 500–507. 

- [19] J. Wang, G. Zhao, H. Xu, Y. Zhao, X. Yang, and H. Huang, “TRUST: Real-time request updating with elastic resource provisioning in clouds,” in _Proc. IEEE Conf. Comput. Commun._ , 2022, pp. 620–629. 

- [20] T. Jia, P. Chen, L. Yang, Y. Li, F. Meng, and J. Xu, “An approach for anomaly diagnosis based on hybrid graph model with logs for distributed services,” in _Proc. IEEE Int. Conf. Web Serv._ , 2017, pp. 25–32. 

- [21] A. Nandi, A. Mandal, S. Atreja, G. B. Dasgupta, and S. Bhattacharya, “Anomaly detection using program control flow graph mining from execution logs,” in _Proc. 22nd ACM SIGKDD Int. Conf. Knowl. Discov. Data Mining_ , 2016, pp. 215–224. 

Authorized licensed use limited to: QUAID E AZAM UNIVERSITY. Downloaded on August 11,2026 at 11:27:28 UTC from IEEE Xplore.  Restrictions apply. 

616 

IEEE TRANSACTIONS ON SERVICES COMPUTING, VOL. 17, NO. 2, MARCH/APRIL 2024 

- [22] G. Yu et al., “MicroRank: End-to-end latency issue localization with extended spectrum analysis in microservice environments,” in _Proc. Web Conf._ , 2021, pp. 3087–3098. 

- [23] G. Yu, Z. Huang, and P. Chen, “TraceRank: Abnormal service localization with dis-aggregated end-to-end tracing data in cloud native systems,” _J. Softw. Evol. Process_ , vol. 35, 2021, Art. no. e2413. 

- [24] H. Mi, H. Wang, Y. Zhou, M. R.-T. Lyu, and H. Cai, “Toward finegrained, unsupervised, scalable performance diagnosis for production cloud computing systems,” _IEEE Trans. Parallel Distrib. Syst._ , vol. 24, no. 6, pp. 1245–1255, Jun. 2013. 

- [25] Z. Li et al., “Practical root cause localization for microservice systems via trace analysis,” in _Proc. IEEE/ACM 29th Int. Symp. Qual. Service_ , 2021, pp. 1–10. 



**Jian Wang** (Member, IEEE) received the PhD degree in computer science from Wuhan University, China, in 2008. He is currently an associate professor with the School of Computer Science, Wuhan University, China. His current research interests include services computing and software engineering. He is now a senior member of the China Computer Federation (CCF), and a member of the CCF Technical Committee on Services Computing (TCSC). 

- [26] Z. Zhang, B. Li, J. Wang, and Y. Liu, “AAMR: Automated anomalous microservice ranking in cloud-native environment,” in _Proc. Int. Conf. Softw. Eng. Knowl. Eng._ , 2021, pp. 86–91. 

- [27] L. Wu, J. Tordsson, E. Elmroth, and O. Kao, “MicroRCA: Root cause localization of performance issues in microservices,” in _Proc. IEEE/IFIP Netw. Operations Manage. Symp._ , 2020, pp. 1–9. 

- [28] Y. Meng et al., “Localizing failure root causes in a microservice through causality inference,” in _Proc. IEEE/ACM 28th Int. Symp. Qual. Service_ , 2020, pp. 1–10. 

- [29] D. Balla, C. Simon, and M. Maliosz, “Adaptive scaling of kubernetes pods,” in _Proc. IEEE/IFIP Netw. Operations Manage. Symp._ , 2020, pp. 1–5. 

- [30] E. Casalicchio and V. Perciballi, “Auto-scaling of containers: The impact of relative and absolute metrics,” in _Proc. IEEE 2nd Int. Workshops Found. Appl. Self* Syst._ , 2017, pp. 207–214. 



**Bing Li** (Member, IEEE) received the PhD degree from the Huazhong University of Science and Technology, Wuhan, China, in 2003. He is currently a professor with the School of Computer Science, Wuhan University. His main research areas are services computing, software engineering, artificial intelligence, and cloud computing. He is now a distinguished member of the China Computer Federation (CCF). 

- [31] M. R. Hossen and M. A. Islam, “A lightweight workload-aware microservices autoscaling with QoS assurance,” 2022, _arXiv:2202.00057_ . 

- [32] L. Baresi and G. Quattrocchi, “A simulation-based comparison between industrial autoscaling solutions and COCOs for cloud applications,” in _Proc. IEEE Int. Conf. Web Serv._ , 2020, pp. 94–101. 

- [33] J. Tong, M. Wei, M. Pan, and Y. Yu, “A holistic auto-scaling algorithm for multi-service applications based on balanced queuing network,” in _Proc. IEEE Int. Conf. Web Serv._ , 2021, pp. 531–540. 

- [34] A. U. Gias, G. Casale, and M. Woodside, “ATOM: Model-driven autoscalingformicroservices,”in _Proc.IEEE39thInt.Conf.Distrib.Comput.Syst._ , 2019, pp. 1994–2004. 

- [35] Z. Yang, P. Nguyen, H. Jin, and K. Nahrstedt, “MIRAS: Model-based reinforcement learning for microservice resource allocation over scientific workflows,” in _Proc. IEEE 39th Int. Conf. Distrib. Comput. Syst._ , 2019, pp. 122–132. 

- [36] X. Zhou et al., “Latent error prediction and fault localization for microservice applications by learning from system trace logs,” in _Proc. 27th ACM Joint Meeting Eur. Softw. Eng. Conf. Symp. Found. Softw. Eng._ , 2019, pp. 683–694. 

- [37] J. Hu, Y. Han, and J. Hu, “Topological potential: Modeling node importance with activity and local effect in complex networks,” in _Proc. 2nd Int. Conf. Comput. Model. Simul._ , 2010, pp. 411–415. 

- [38] Y. Mao, L. Zhou, and N. Xiong, “TPS: A topological potential scheme to predict influential network nodes for intelligent communication in social networks,” _IEEE Trans. Netw. Sci. Eng._ , vol. 8, no. 1, pp. 529–540, First Quarter 2021. 

- [39] G. Jeh and J. Widom, “Scaling personalized web search,” in _Proc. 12th Int. Conf. World Wide Web_ , 2003, pp. 271–279. 





**Zekun Zhang** received the MS degree from the School of Computer Science, Wuhan University, China. He iscurrently working toward thePhDdegree with the School of Computer Science, Wuhan University. His research interests include cloud computing and fault diagnosis. 

**Duantengchuan Li** is currently working toward the PhD degree in software engineering with the School of Computer Science, Wuhan University, Wuhan, China. His research interests include recommendation system, representation learning, natural language processing, intention recognition, pattern recognition, computer vision and their applications in software engineering, intelligent education, and intelligent sports. 

- [40] O. Keyes, “Wiki-pageviews, English wikipedia pageviews by second,” 2015. [Online]. Available: http://datahub.io/dataset/english-wikipediapageviews-by-second 

- [41] M. Abdullah, W. Iqbal, J. L. Berral, J. Polo, and D. Carrera, “Burst-aware predictive autoscaling for containerized microservices,” _IEEE Trans. Services Comput._ , vol. 15, no. 3, pp. 1448–1460, May/Jun. 2022. 

- [42] Z. Ding and Q. Huang, “COPA: A combined autoscaling method for kubernetes,” in _Proc. IEEE Int. Conf. Web Serv._ , 2021, pp. 416–425. 

**Shuaiyu Xie** received the BS degree from the School of Computer Science, Wuhan University, China, in 2021. He is currently working toward the PhD degree with Wuhan University. His current research interests include services computing, artificial intelligence for IT operations (AIOps), and task scheduling. 



**Patrick C. K. Hung** received the bachelor’s degree in computer science from the University of New South Wales, Sydney, NSW, Australia, in 1993, the master’s and PhD degrees in computer science from the Hong Kong University of Science and Technology, Hong Kong, in 1995 and 2001, respectively, and the master’s degree in management sciences from the University of Waterloo, Canada, in 2002. He is a professor and the director of International Programs with the Faculty of Business and Information Technology, Ontario Tech University, Canada. His research interests 



includeservicescomputing,cloudcomputing,BigData,andedgecomputing.He is a founding member of the IEEE Technical Committee on Services Computing and the _IEEE Transactions on Services Computing_ . 

Authorized licensed use limited to: QUAID E AZAM UNIVERSITY. Downloaded on August 11,2026 at 11:27:28 UTC from IEEE Xplore.  Restrictions apply. 

