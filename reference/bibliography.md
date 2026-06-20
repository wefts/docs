# Бібліографія — Heterogeneous Cognitive Swarm

<!-- markdownlint-disable MD029 -->

*Анотований список першоджерел. Кожна позиція має короткий рядок «**→ нам:**» —
навіщо вона проєкту. Мета — спиратися на перевірені (емпіричні) або хоча б добре
описані (теоретичні) тези.*

**Умовні позначки надійності цитати:**
`[✓]` — вихідні дані звірені під час дослідження;
`[~]` — загальновідома класика, дані з пам'яті, варто звірити рік/сторінки перед
формальним посиланням;
`[?]` — точні вихідні дані потребують перевірки.

---

## 1. Фундамент computer science

1. **Turing, A. M. (1936).** On Computable Numbers, with an Application to the
   Entscheidungsproblem. *Proceedings of the London Mathematical Society*,
   s2-42(1), 230–265. `[~]`
   **→ нам:** означення обчислюваності та межі алгоритмічного; філософська рамка
   «що взагалі може робити машина».

2. **Shannon, C. E. (1948).** A Mathematical Theory of Communication. *Bell System
   Technical Journal*, 27, 379–423, 623–656. `[~]`
   **→ нам:** інформація як лог-міра несподіванки — спільний корінь із нашим
   confidence calculus; предок loss-функції.

3. **Rosenblatt, F. (1958).** The Perceptron: A Probabilistic Model for
   Information Storage and Organization in the Brain. *Psychological Review*,
   65(6), 386–408. `[~]`
   **→ нам:** перша машина, що вчиться; історичний корінь Python-стовпа.

4. **Minsky, M. & Papert, S. (1969).** *Perceptrons: An Introduction to
   Computational Geometry*. MIT Press. `[~]`
   **→ нам:** межі простих моделей і чому потрібні шари; урок про чесну оцінку
   можливостей.

5. **Lamport, L. (1978).** Time, Clocks, and the Ordering of Events in a
   Distributed System. *Communications of the ACM*, 21(7), 558–565. `[~]`
   **→ нам:** несуча теза стовпа Elixir/OTP — причинне впорядкування подій,
   логічні годинники. Найважче-ретрофітна частина координації.

6. **Rumelhart, D. E., Hinton, G. E. & Williams, R. J. (1986).** Learning
   representations by back-propagating errors. *Nature*, 323(6088), 533–536. `[~]`
   **→ нам:** як навчати глибокі мережі; контекст для embedding/inference-сайдкара.

7. **Brin, S. & Page, L. (1998).** The Anatomy of a Large-Scale Hypertextual Web
   Search Engine. *Computer Networks and ISDN Systems*, 30(1–7), 107–117. `[~]`
   **→ нам:** PageRank як «голос, зважений довірою голосувальника» — рання модель
   впевненості через структуру графа, не через частоту.

8. **Krizhevsky, A., Sutskever, I. & Hinton, G. E. (2012).** ImageNet
   Classification with Deep Convolutional Neural Networks. *NeurIPS 25*. `[~]`
   **→ нам:** момент, коли «дані + обчислення + архітектура» зробили глибоке
   навчання робочим.

9. **Vaswani, A., et al. (2017).** Attention Is All You Need. *NeurIPS 30*. `[~]`
   **→ нам:** трансформер — субстрат великих моделей, які ми кличемо рідко й
   навмисно.

10. **Brown, T. B., et al. (2020).** Language Models are Few-Shot Learners.
    *NeurIPS 33*. `[~]`
    **→ нам:** теза «інтелект емерджентний на масштабі» — філософська
    протилежність, від якої ми свідомо відштовхуємось (cost asymmetry).

---

## 2. Розподілені системи та патерни архітектури

11. **Lamport, L. (1998).** The Part-Time Parliament. *ACM TOCS*, 16(2),
    133–169. `[~]`
    **→ нам:** Paxos — консенсус між вузлами; основа виборів лідера в ядрі.

12. **Grassé, P.-P. (1959).** La reconstruction du nid et les
    coordinations interindividuelles… La théorie de la stigmergie.
    *Insectes Sociaux*, 6, 41–80. `[?]`
    **→ нам:** першоджерело **стигмергії** — координація через сліди в середовищі.

13. **Hayes-Roth, B. (1985).** A blackboard architecture for control. *Artificial
    Intelligence*, 26(3), 251–321. `[~]`
    **→ нам:** канонічний **blackboard-патерн** — програмний родич стигмергії;
    граф = дошка, воркери = knowledge sources.

14. **Cockburn, A. (2005).** Hexagonal Architecture (Ports and Adapters). `[~]`
    **→ нам:** першоджерело «портів і адаптерів»; стиль ядра + плагінів.

15. **Fowler, M. (2005).** *Event Sourcing* (martinfowler.com). `[~]`
    **→ нам:** append-only журнал як **антидот реконсолідаційного дрейфу**;
    ідіоматично для Elixir.

---

## 3. Нейронаука пам'яті (ядро лінії «б»)

16. **Hebb, D. O. (1949).** *The Organization of Behavior*. Wiley. `[~]`
    **→ нам:** «fire together, wire together» — прообраз підсилення ребер.

17. **Nader, K., Schafe, G. E. & LeDoux, J. E. (2000).** Fear memories require
    protein synthesis in the amygdala for reconsolidation after retrieval.
    *Nature*, 406(6797), 722–726. `[✓]`
    **→ нам:** канонічний доказ **реконсолідації** — слід при згадуванні знову стає
    нестабільним. Біологічний шаблон головної загрози нашої traversal-петлі.

18. **Nader, K., Schafe, G. E. & LeDoux, J. E. (2000).** The labile nature of
    consolidation theory. *Nature Reviews Neuroscience*, 1(3), 216–219. `[✓]`
    **→ нам:** оглядова рамка до п.17.

19. **Debiec, J., Doyère, V., Nader, K. & LeDoux, J. E. (2006).** Directly
    reactivated, but not indirectly reactivated, memories undergo reconsolidation
    in the amygdala. *PNAS*, 103(9), 3428–3433. `[✓]`
    **→ нам:** ключовий нюанс — лабільним стає лише **прямо** реактивований слід.
    Дизайн-натяк: не все, що зачепив обхід, має ставати «м'яким».

20. **Loftus, E. F. (2005).** Planting misinformation in the human mind: A 30-year
    investigation of the malleability of memory. *Learning & Memory*, 12(4),
    361–366. `[~]`
    **→ нам:** емпірика реконструктивності — пам'ять переписується тим, ким ти є
    зараз.

21. **Squire, L. R., Genzel, L., Wixted, J. T. & Morris, R. G. (2015).** Memory
    consolidation. *Cold Spring Harbor Perspectives in Biology*, 7(8), a021766. `[✓]`
    **→ нам:** сучасний огляд консолідації; модель для фонового воркера-консолідатора
    (sleep replay).

---

## 4. Нейронаука кортексу та Thousand Brains

22. **Mountcastle, V. B. (1997).** The columnar organization of the neocortex.
    *Brain*, 120(4), 701–722. `[✓]`
    **→ нам:** першоджерело **колонкової організації** — кора як повторення одного
    модуля. Біологічна підтримка «системи з однотипних частин».

23. **Hubel, D. H. & Wiesel, T. N. (1962).** Receptive fields, binocular
    interaction and functional architecture in the cat's visual cortex. *Journal
    of Physiology*, 160(1), 106–154. `[~]`
    **→ нам:** підтвердження колонкового принципу в зоровій корі.

24. **Hawkins, J. & Ahmad, S. (2016).** Why Neurons Have Thousands of
    Synapses, a Theory of Sequence Memory in Neocortex. *Frontiers in Neural
    Circuits*, 10:23. `[✓]`
    **→ нам:** механізм передбачення/послідовностей у колонці.

25. **Hawkins, J., Lewis, M., Klukas, M., Purdy, S. & Ahmad, S. (2019).** A
    Framework for Intelligence and Cortical Function Based on Grid Cells in the
    Neocortex. *Frontiers in Neural Circuits*, 12:121. `[~]`
    **→ нам:** ґрид-клітини й референтні рамки — прообраз «ознака → локація» як
    типізованого ребра.

26. **Hawkins, J. (2021).** *A Thousand Brains: A New Theory of Intelligence*.
    Basic Books. (передмова Р. Докінза). `[~]`
    **→ нам:** доступний виклад теорії «150 000 міні-мозків» і голосування.
    ⚠ У відео-описі Докінза помилково названо співавтором — він автор передмови.

27. **Harris, K. D. & Shepherd, G. M. G. (2015).** The neocortical circuit: themes
    and variations. *Nature Neuroscience*, 18(2), 170–181. `[~]`
    **→ нам:** шестишарова мікросхема кори — анатомічна основа TBT.

---

## 5. Мультиагентні системи та емерджентність

28. **Park, J. S., O'Brien, J. C., Cai, C. J., Morris, M. R., Liang, P. &
    Bernstein, M. S. (2023).** Generative Agents: Interactive Simulacra of Human
    Behavior. *UIST 2023*. arXiv:2304.03442. `[✓]`
    **→ нам:** Smallville — емерджентна поведінка з мінімального опису.

29. **Altera.AL, et al. (2024).** Project Sid: Many-agent simulations toward AI
    civilization. arXiv:2411.00114. `[✓]`
    **→ нам:** 1000 агентів у Minecraft — емпіричний доказ **неконтрольованої
    пропагації** при прямій комунікації; обґрунтування нашого вибору стигмергії.

30. **Qian, C., et al. (2023).** ChatDev: Communicative Agents for Software
    Development. arXiv:2307.07924. `[✓]`
    **→ нам:** ієрархія агентів-ролей (CEO/CTO/...); пор. gate + consilium, але з
    прямою комунікацією.

---

## 6. Колапс, рекурсивні петлі, виведення на графах

31. **Shumailov, I., Shumaylov, Z., Zhao, Y., Papernot, N., Anderson, R. & Gal, Y.
    (2024).** AI models collapse when trained on recursively generated data.
    *Nature*, 631(8022), 755–759. `[✓]`
    **→ нам:** найчистіший CS-аналог дрейфу — рекурсія без зовнішнього якоря →
    зникнення хвостів, необоротний колапс.

32. **Pearl, J. (1988).** *Probabilistic Reasoning in Intelligent Systems:
    Networks of Plausible Inference*. Morgan Kaufmann. `[~]`
    **→ нам:** першоджерело noisy-OR і belief propagation — фундамент нашого
    confidence calculus.

33. **Murphy, K. P., Weiss, Y. & Jordan, M. I. (1999).** Loopy belief propagation
    for approximate inference: An empirical study. *UAI '99*. `[✓]`
    **→ нам:** BP на графах із циклами працює на практиці, але без гарантій.

34. **Weiss, Y. (2000).** Correctness of local probability propagation in graphical
    models with loops. *Neural Computation*, 12(1), 1–41. `[~]`
    **→ нам:** першоджерело про **подвійний облік / надмірну впевненість** у циклах
    — формальна назва нашої проблеми незалежності шляхів.

35. **Ihler, A. T., Fisher, J. W. & Willsky, A. S. (2005).** Loopy belief
    propagation: Convergence and effects of message errors. *JMLR*, 6, 905–936. `[✓]`
    **→ нам:** кількісно про переоблік і помилки повідомлень.

36. **Yedidia, J. S., Freeman, W. T. & Weiss, Y. (2005).** Constructing
    free-energy approximations and generalized belief propagation algorithms.
    *IEEE Transactions on Information Theory*, 51(7), 2282–2312. `[✓]`
    (також рання версія: *Generalized Belief Propagation*, NeurIPS 13, 2000.)
    **→ нам:** **принциповий** спосіб коригувати подвійний облік (GBP на регіонах)
    — простір рішень для «дешевого виявлення незалежності шляхів».

37. **Mooij, J. M. & Kappen, H. J. (2007).** Loop corrections for approximate
    inference on factor graphs. *JMLR*, 8, 1113–1143. `[✓]`
    **→ нам:** альтернативний клас loop-corrected методів.

---

## 7. Пам'ять LLM-агентів (сучасна лінія, 2023–2026)

38. **Packer, C., et al. (2023).** MemGPT: Towards LLMs as Operating Systems.
    arXiv:2310.08560. `[✓]`
    **→ нам:** пам'ять як керована ОС-подібна ієрархія; рання рамка.

39. **Gutiérrez, B. J., et al. (2024).** HippoRAG: Neurobiologically Inspired
    Long-Term Memory for Large Language Models. *NeurIPS 2024*. `[~]`
    **→ нам:** індексування пам'яті за **гіпокампальною теорією** — прямий міст від
    нейронауки до інженерії графа.

40. **Xu, W., Liang, Z., Mei, K., Gao, H., Tan, J. & Zhang, Y. (2025).** A-Mem:
    Agentic Memory for LLM Agents. arXiv:2502.12110. `[✓]`
    **→ нам:** self-reflective оновлення пам'яті; критика: оновлює за лічильником
    токенів, а не за семантичною завершеністю.

41. **(Огляд) Rethinking Memory in LLM-based Agents: Representations, Operations,
    and Emerging Topics (2025).** arXiv:2505.00675. `[✓]`
    **→ нам:** карта простору пам'яті агентів (індексація graph/signal/timeline,
    операції оновлення); згадує епізодичну «what-where-when».

42. **(Каркас SSGM) Governing Evolving Memory in LLM Agents: … Stability and
    Safety Governed Memory (2026).** arXiv:2603.11768. `[✓]`
    **→ нам:** три точки відмови (отруєння / семантичний дрейф / конфлікт при
    retrieval); антидот — розв'язати еволюцію пам'яті від виконання, decay-модель,
    контроль доступу до консолідації. Дає ім'я нашій lambda: stability-plasticity.

43. **(GAM) Hierarchical Graph-based Agentic Memory for LLM Agents (2026).**
    arXiv:2604.12285. `[✓]`
    **→ нам:** **ізоляція запису** проти контамінації/дрейфу; графова ієрархічна
    пам'ять — близько до нашої моделі.

44. **(Огляд безпеки) A Survey on the Security of Long-Term Memory in LLM Agents:
    Toward Mnemonic Sovereignty (2026).** arXiv:2604.16548. `[✓]`
    **→ нам:** прямо вживає «reconsolidated / drift»; пропонує **pre-consolidation
    gate** з перевіркою provenance і консистентності; мультиагентна контамінація
    через спільне сховище — перетин із нашою privacy-загрозовою моделлю.

---

## Прогалини, які варто закрити далі

- **Конкретні дані з арабського ряду нейробіології** (`[~]`/`[?]`) звірити перед
  формальним цитуванням: Grassé 1959, Weiss 2000, Hubel & Wiesel 1962.
- **Кількісне вимірювання дрейфу** в агентній пам'яті (бенчмарки LongMemEval,
  LoCoMo) — окрема гілка, якщо захочемо мірити, а не лише описувати.
- **Формальна збіжність стигмергічних петель** (feedback-loop stability) — поки
  лежить між теорією керування й loopy-BP; першоджерела ще не зафіксовані.
- **Thousand Brains Project (Clay et al. 2024; Hawkins, Leadholm, Clay 2025)** —
  інженерна реалізація TBT; додати, коли візьмемося за гілку «а».
