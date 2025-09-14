# Splitting Datasets

- [binary split](https://github.com/i-krishna/Datawarehouse-Datamining-OLAP/blob/master/Data%20Mining%20Privacy/Confusion%20matrix%20binary%20classification.png) - train, test - [Spam-email-filter binary classification code](https://github.com/i-krishna/Business-Analytics/blob/main/Data-Science/Python/spam-email-filter.py) 

- [Multi split code](https://github.com/i-krishna/Business-Analytics/blob/main/Data-Science/Python/train_llm.py) - train, validate after each layer / epoch, test

https://developers.google.com/machine-learning/crash-course/classification/accuracy-precision-recall
```
Precision = TP / (TP + FP)
Recall = TP / (TP + FN)
Accuracy = (TP + TN) / (TP + TN + FP + FN)

Where,
TP - True +ve
FP - False +ve
FP - False +ve
FN - False -ve
```

# Ethics & Bias across 6 phases of CRISP-DM 2.0
Sept 7, 2025 by Krishna Damarla

[Balancing skewed data](https://github.com/i-krishna/Business-Analytics/blob/main/Data-Science/Python/balance-skewed-data.py) is a critical step in machine learning to prevent models from becoming biased toward the majority class. For classification tasks, this is called handling class imbalance. For regression, it addresses uneven distributions in the target variable. 

- Oversampling is a technique that increases the number of instances in the minority class to match the number of instances in the majority class. This creates a more balanced training dataset, preventing the model from ignoring the minority class.
- Undersampling is a technique that reduces the number of instances in the majority class to balance the class distribution. 

To handle bias further, build your AI Agent or LLM by integrating explainable AI (XAI) frameworks like IBM’s AI Explainability 360 [AIX360](https://github.com/Trusted-AI/AIX360) or others from this [list](https://securing.ai/ai-security/explainable-ai-frameworks/) of Securing AI. Adding explainable AI frameworks helps everyone see how models work. For example, explainable AI framework could explain why the model flagged a [loan application](https://github.com/Trusted-AI/AIX360/blob/master/examples/tutorials/HELOC.ipynb). Such a reasoning built trusts, confidence in decisions and also meets legal requirements such as GDPR. Such an XAI integration into models makes the model trusthworthy for users to ask questions and get trusted answers.

XAI goes beyond basic interpretability by providing clear, understandable explanations for complex models like deep neural networks and ensemble methods. While traditional interpretability works well for simple prediction models (like decision trees or linear regression), XAI uses advanced techniques (such as SHAP, LIME, counterfactuals, and visualizations) to explain predictions from “black box” models, making AI decisions transparent and trustworthy for users and stakeholders.

Basicc interpretability in prediction models:

```
Decision Trees:
You can trace the path from root to leaf to see exactly how a prediction was made (e.g., “If age > 30 and income > $50K, then approve loan”).

Linear Regression:
The coefficients show how each feature affects the prediction (e.g., “Each extra year of education increases salary by $2,000”).

Logistic Regression:
You can see the odds ratio for each variable, explaining how it influences the probability of an outcome.

Feature Importance in Random Forests:
The model ranks features by how much they contribute to predictions, helping you understand which factors matter most.

SHAP Values:
SHAP (SHapley Additive exPlanations) values show how each feature contributed to a specific prediction for an individual data point.
```

[Link to Report](https://github.com/i-krishna/Datawarehouse-Datamining-OLAP/blob/master/Data%20Mining%20Privacy/CRISP-DM%20and%20Laws%20of%20Data%20Mining%20Review.pdf )

# A Simple Take on Cybersecurity
by Krishna Damarla, 10-Sept-2025 

Cyber threats keep rising every year. So, organizations need more than one wall of defense. They need layers, like a castle with gates, guards, and secret tunnels.

I saw this play out in one of my federal banking projects. We started simple. Employees had to tap their ID cards just to unlock their laptops. Only then could they reach sensitive databases. On top of that, we built in end-to-end encryption and regular penetration tests. The project also enforced multi-factor authentication (MFA), regular rotation of keys and passwords, and a Zero Trust approach, meaning every request to touch sensitive data had to prove itself again and again to protect data privacy (ASIS International, 2008). 

We did not stop there. Firewalls, intrusion detection systems, and frequent audits became part of the routine. These steps were not just technical boxes to check. They are grounded on the principle of “do no harm, act with integrity”. Also, they lined up with EU's GDPR and U.S California Consumer Privacy Act data protection laws demanding strong safeguards to protect consumer data. 

The Principle of Least Privilege (PoLP) of “Give people only what they need to do their job” is strictly followed in the project, giving users access to exactly what they need and nothing more. I remember a junior analyst on that project. Even after passing the smart card check, he could only open the specific datasets tied to his task. Not the whole customer database. That one simple guardrail shrank the attack surface. It also kept personal data safe from misuse (IEEE, n.d.).

PoLP is more than policy. It is respect in action, respect for privacy, and respect for autonomy. It also shows regulators that the company takes due diligence seriously. That means stronger compliance plus fewer risks of heavy penalties. Security can feel like a maze of tech buzzwords. But when you break it down, it is just about trust. Protecting people’s data, the way you would protect your own.

**References**

ASIS International. (2008). Can data mining and privacy coexist? Security Management. https://www.asisonline.org/security-management-magazine/articles/2008/11/can-data-mining-and-privacy-coexist/

Newton, E. (2023). Top 5 challenges in ethical data mining we need to overcome. Datafloq. https://datafloq.com/read/top-5-challenges-ethical-data-mining-need-overcome/ 

# AI & ML: Legal Requirements and Global Variations
Aug 27, 2025 by Krishna Damarla

Organizations implementing data mining, Machine Learning (ML), and AI (Artificial Intelligence) algorithms across different regions often face legal problems because of the different data privacy rules and ethical standards across geographies (IEEE, n.d.).

GDPR vs. US Data Privacy Laws

The European Union (EU)’s GDPR is very detailed and applies strict rules to any company that handles EU residents data. GDPR focuses on protecting individual rights, limiting the amount of data used, and holding companies accountable (Da Bormida, 2021). Whereas U.S. laws are less uniform and are based on specific industries or states.

For example, HIPAA protects health information, and FERPA protects student records (CITI Program Staff, 2024). The California Consumer Privacy Act protects the consumer rights of State of California residents. Only around 20 out of 50 U.S. states have enacted detailed data privacy laws (Bloomberg Law, n.d.). These laws often include more exceptions and a balance between privacy and business operations. Because of this, U.S. regulations usually give companies more flexibility (IEEE, n.d.).

Legal Challenges 

Global companies that use data mining, ML, and AI face complicated legal situations because there are many overlapping and sometimes conflicting rules. The GDPR usually imposes more stricter rules on companies than many U.S. federal or industry-specific laws, making it the toughest standard to follow (Da Bormida, 2021).

When it comes from third parties, it can be hard to ensure data is collected properly, especially if their consent practices are unclear or depend on the country they’re in and what is legally allowed versus what people consider ethical, especially in cases like emergency responses or public health (IEEE, n.d.). Also, laws like new state privacy acts, moving data internationally, and AI governance rules keep changing. 

Overcoming the Challenges

Companies should consider a single data governance system that follows the strictest applicable rules, such as GDPR (Da Bormida, 2021) and they should always stay updated to quickly adjust their procedures to overcome the legal challenges.

They should implement methods such as privacy by design, pseudonymization, data minimization, anonymization, and managing user consent by sending notification alerts to users to update their consent preferences. Further, integrating regulatory approval workflows can create a system that enables responsible use of data that respects people's rights while still supporting innovation and public good (Da Bormida, 2021; IEEE, n.d.).

References

Bloomberg Law. (n.d.). Which states have consumer data privacy laws? Bloomberg Law State Privacy Legislation Tracker. https://pro.bloomberglaw.com/insights/privacy/state-privacy-legislation-tracker/

CITI Program Staff. (2024). How HIPAA and FERPA rules affect school-based health clinics. CITI Program. https://about.citiprogram.org/blog/how-hipaa-and-ferpa-rules-affect-school-based-health-clinics/ 

Da Bormida, M. (2021). The Big Data World: Benefits, Threats and Ethical Challenges. Ethical issues in covert, security and surveillance research (Vol. 8). Emerald Publishing. https://doi.org/10.1108/S2398-601820210000008007

IEEE. (n.d.). Ethical issues related to data privacy and security: Why we must balance ethical and legal requirements in the connected world. IEEE Digital Privacy https://digitalprivacy.ieee.org/publications/topics/ethical-issues-related-to-data-privacy-and-security-why-we-must-balance-ethical-and-legal-requirements-in-the-connected-world/

<img width="890" height="210" alt="image" src="https://github.com/user-attachments/assets/e23ec175-3331-448b-90e2-9c3aee3aaff9" />


# Ethical Data Mining Challenges & Solutions in the age of LLMs & AI Agents
Aug 20, 2025 by Krishna Damarla

Data mining has changed a lot since IBM's supercomputer Watson, based on DeepQA architecture won  Jeopardy show in 2011 (Dow, n.d.) and DeepBlue chess playing supercomputer defeated the world chess champion in 1997 (Delen, 2020). Various advancements have taken place in the fields of machine learning, natural language processing, deep learning, big data & data science, which have taken data mining far beyond its early days. Today’s AI systems can solve complex problems across various fields.

Right now, data mining brings up important ethical issues, especially as large language models (LLMs) and AI agents play a huge role in making decisions (Newton, 2023). Main challenges include bias, lack of transparency, poor consent processes, and misuse, all made worse by the scale and independence of modern AI systems, Multi-Agents and Agent-enabled-Agents.

Bias is a big problem. Models trained on historical data can increase social prejudices (Da Bormida, 2021). Automated decisions in credit scoring, legal systems and hiring, can lead to unfair results, especially if models are used without regular bias checks and not on diverse training datasets. In the LLM age, bias can spread quickly through generated content and automated suggestions, affecting millions of people worldwide.

Transparency is also a challenge because deep learning models are complex. LLMs and AI agents often work like "black boxes," making it hard to explain or question their decisions (Boujemaa, 2018). Laws like GDPR require clear explanations (Boujemaa, 2018) to build trust in these models. New methods like Explainable AI (XAI) and model cards for LLMs to be considered. And companies must ensure they have robust data governance policies to hold AI models accountable. Consent models are not up to date. People often don’t know how their data is used by AI agents (Newton, 2023) and the risk of misuse grows as models use data for unforeseen purposes (Da Bormida, 2021) such as multi-level marketing (MLM).

Solutions include data minimization, user-centric data stores, and dynamic consent mechanisms tailored for AI-driven environments. Also, integrating governance, risk, and compliance management into AI models is important (Newton, 2023). Ethics, economics, law, and social impact (rather than just coding) to be considered in the model design process (co-creation), as most algorithms are now open-source and reused.

In conclusion, ethical data mining in the age of LLMs and AI agents needs strong technical implementation integrated with robust governance principles, protection, and a shift towards fairness, transparency, and respect for human dignity.

**References**

Boujemaa, N. (2018). Transparency and accountability of algorithmic systems vs. GDPR?. The Alan Turing Institute. https://www.youtube.com/watch?v=xiSI-Xgrtpo  

Da Bormida, M. (2021). The big data world: Benefits, threats and ethical challenges. In R. Iphofen & D. O'Mathúna (Eds.), Ethical issues in covert, security and surveillance research (Vol. 8, Chapter 6). Emerald Publishing Limited. https://doi.org/10.1108/S2398-601820210000008007

Delen, D. (2020). Predictive analytics: Data mining, machine learning, and data science for practitioners (2nd ed.). Pearson.

Dow, E. M. (n.d.). A hardware & software overview. IBM. [https://research.ibm.com/publications?source=19579 ](https://share.confex.com/share/117/webprogram/Handout/Session9577/WatsonHardwareAndSoftware.pdf) & https://research.ibm.com/publications/building-watson-an-overview-of-the-deepqa-project 

Newton, E. (2023). Top 5 challenges in ethical data mining we need to overcome. https://datafloq.com/read/top-5-challenges-ethical-data-mining-need-overcome/

https://research.ibm.com/publications?source=19579 
