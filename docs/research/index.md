# Academic Research & Contributions

My research bridges the gap between theoretical verification and applied engineering, focusing on autonomous systems and neural network robustness. 

---

## :material-book-open-page-variant: Publications & Case Studies

!!! abstract "Proving Termination and Correctness of the VeRAPAk Neural Network Verifier in Dafny (In Preparation)"
    **Venue:** FMCAD 2026 | **Authors:** **M. Davis**, et al.

    **Overview:** A case study detailing the rigorous mathematical grounding of the VeRAPAk neural network framework to guarantee system correctness.

    * **My Contribution:** Formally modeled the Python-based framework in Dafny to generate computer-assisted proofs, Contract-aware fuzzing for analysis of Dafny proofs applied on Python concrete execution
    * **Tech Stack:** `Dafny` `Python` `Docker` `Atheris`

    [View Draft (PDF)](#){ .md-button .md-button--small } [Artifact](#){ .md-button .md-button--small }


!!! abstract "2024 | Identification and Localization of Wind Turbine Blade Faults Using Deep Learning"
    **Venue:** Applied Sciences | **Authors:** **M. Davis**, et al.

    **Overview:** This study evaluates deep learning methodologies for detecting structural faults on wind turbine blades, providing a comparative analysis of single-stage (YOLO) and two-stage (Mask R-CNN) object detection architectures.

    * **My Contribution:** To address the scarcity of commercial fault data, I architected a custom computer vision data pipeline using CVAT to curate and label the 6,000-image CAI-SWTB dataset. Using PyTorch, I then developed, trained, and evaluated multiple YOLO models (v5, v8, v9), ultimately establishing YOLOv9 C as the optimal localization architecture for the task with an mAP50 of 0.849.
    * **Tech Stack:** `PyTorch` `YOLO` `CVAT` `Python`

    [Read Full Paper](/assets/publications/IdentificationandLocalization-Davis-2024.pdf){ .md-button .md-button--small }

!!! abstract "2024 | Deep Learning for Indoor Pedestal Fan Blade Inspection"
    **Venue:** Drones | **Authors:** A. Rodriguez, **M. Davis**, et al.

    **Overview:** This study introduces a low-cost, drone-based educational framework to simulate wind turbine inspections, evaluating various deep learning models on pedestal fan blades to teach automated fault detection.

    * **My Contribution:** I co-authored the manuscript and led the deep learning experiments to evaluate the system's predictive maintenance capabilities. I trained and analyzed multiple architectures (ViT, DenseNet, Xception, ResNet), demonstrating that the ViT and DenseNet models achieved 100% accuracy in classifying structural anomalies.
    * **Tech Stack:** `Python` `ViT` `DenseNet` `Xception` `ResNet` `Tello SDK`
    
    [Read Full Paper](/assets/publications/drones-Davis-2024.pdf){ .md-button .md-button--small }

!!! abstract "2024 | Anomaly Detection on Small Wind Turbine Blades"
    **Venue:** Energies | **Authors:** B. Altice, E. Nazario, **M. Davis**, et al.

    **Overview:** This study explores the application of various deep learning architectures—including Xception, VGG19, ResNet50, AlexNet, and a custom Convolutional Neural Network (CNN)—to accurately detect structural anomalies such as erosion, holes, and cracks on small-scale wind turbine blades.

    * **My Contribution:** I conducted formal analysis, validation, and investigation of the deep learning models to ensure the reliability of the anomaly detection results. Additionally, I contributed to the data visualization and the technical review and editing of the published manuscript.
    * **Tech Stack:** `Python` `Xception` `VGG19` `ResNet50` `AlexNet`

    [Read Full Paper](/assets/publications/anomalydetection-davis-2024.pdf){ .md-button .md-button--small }


!!! abstract "2024 | Image Classification of Forest Fires Using Machine and Deep Learning"
    **Venue:** IEEE i-ETC | **Authors:** **M. Davis**, et al.

    **Overview:** This study investigates the efficacy of various machine learning and deep learning algorithms for the automated classification of forest fire imagery to aid in early detection systems.

    * **My Contribution:** I led the experimental design, trained and evaluated the classification architectures, and was responsible for the technical writing and preparation of the manuscript.
    * **Tech Stack:** `Python` `Deep Learning` `Machine Learning`

    [Read Full Paper](/assets/publications/classificationfires-davis-2024.pdf){ .md-button .md-button--small }



!!! abstract "2023 | Desert/Forest Fire Detection Using Machine/Deep Learning Techniques"
    **Venue:** Fire | **Authors:** **M. Davis**, M. Shekaramiz

    **Overview:** This study proposes novel transfer learning approaches for desert and forest fire detection using ResNet-50 and Xception architectures. To address the lack of arid environment representation in existing data, the research introduces the "Utah Desert Fire" dataset, on which the proposed models achieved up to 100% classification accuracy.

    * **My Contribution:** I curated the novel 986-image "Utah Desert Fire" dataset by conducting controlled burn experiments and capturing aerial drone imagery. I developed the software pipeline using TensorFlow and Keras to train, tune, and evaluate multiple architectures—including SVM, MobileViT, ResNet-50, and Xception—and served as the primary author for the manuscript.
    * **Tech Stack:** `Python` `TensorFlow` `Keras` `ResNet-50` `Xception` `MobileViT` `SVM`

    [Read Full Paper](/assets/publications/fire-Davis-2023.pdf){ .md-button .md-button--small }

---

## :material-presentation: Presentations & Posters

!!! info "FMCAD 2025 Student Forum"
    **Topic:** VeRAPAk & Neural Network Robustness
    
    Presented the foundational architecture of VeRAPAk, a neural network verification framework, and a case study using VeRAPAk on the HiggsML Uncertainty Challenge.
    
    [View Poster (PDF)](/assets/publications/FMCAD25-VeRAPAk-Poster.pdf){ .md-button .md-button--small }

!!! info "9th Annual Sandia National Labs ML/DL Workshop"
    **Topic:** VeRAPAk & Neural Network Robustness
    
    Presented the foundational architecture of VeRAPAk, a neural network verification framework, and a case study using VeRAPAk on the HiggsML Uncertainty Challenge.

    [View Workshop Slides (PDF)](/assets/publications/VeRAPAk-SandiaMLDL25.pptx){ .md-button .md-button--small } [View Video Presentation](https://youtu.be/pJ2YFwf3-3c?si){ .md-button .md-button-small}

---

## :material-account-tie: Academic Service & Volunteering

I actively participate in the academic peer-review process, ensuring the reproducibility and technical validity of formal verification research.

* **Artifact Evaluation (AE) Committee Member** | *SPIN 2026*
  > Executed technical smoke tests and reproducibility evaluations for submitted research artifacts, ensuring Dockerized environments and verification tools functioned as claimed by authors. 

* **Peer Reviewer** | *FMCAD 2025*
  > Participated in peer-review evaluations for formal methods and computer-aided design research submissions.

* **Peer Reviewer** | *IEEE i-ETC 2024*
  > Reviewed emerging technology submissions focusing on machine learning and systems engineering.