<script>
    let expression = $state("");
    let display_number = $state("0");
    let is_evaluated = $state(false);
    let is_deg = $state(true); // DEG vs RAD mode for trigonometry

    /**
     * Factorial helper
     * @param {number} n
     * @returns {number}
     */
    function factorial(n) {
        if (n < 0 || !Number.isInteger(n)) return NaN;
        if (n === 0 || n === 1) return 1;
        let res = 1;
        for (let i = 2; i <= n; i++) {
            res *= i;
            if (!Number.isFinite(res)) break;
        }
        return res;
    }

    /**
     * Safely evaluate a mathematical expression
     * @param {string} expr
     * @returns {number}
     */
    function evaluateExpression(expr) {
        let sanitized = expr
            .replace(/×/g, "*")
            .replace(/÷/g, "/")
            .replace(/π/g, `${Math.PI}`)
            .replace(/\be\b/g, `${Math.E}`);

        // Custom parser using Shunting-yard / Tokenizer
        const tokens = tokenize(sanitized);
        const postfix = toPostfix(tokens);
        return evaluatePostfix(postfix);
    }

    /**
     * Tokenize expression string
     * @param {string} str
     * @returns {string[]}
     */
    function tokenize(str) {
        const tokens = [];
        let i = 0;
        const n = str.length;

        while (i < n) {
            const ch = str[i];
            if (/\s/.test(ch)) {
                i++;
                continue;
            }

            if (/\d/.test(ch) || ch === ".") {
                let num = "";
                while (i < n && (/\d/.test(str[i]) || str[i] === ".")) {
                    num += str[i];
                    i++;
                }
                tokens.push(num);
                continue;
            }

            if (/[a-zA-Z]/.test(ch)) {
                let fn = "";
                while (i < n && /[a-zA-Z]/.test(str[i])) {
                    fn += str[i];
                    i++;
                }
                tokens.push(fn);
                continue;
            }

            // Negative sign vs subtraction
            if (ch === "-") {
                const prev = tokens[tokens.length - 1];
                const isUnary = !prev || ["+", "-", "*", "/", "^", "%", "("].includes(prev);
                if (isUnary) {
                    tokens.push("neg");
                    i++;
                    continue;
                }
            }

            tokens.push(ch);
            i++;
        }
        return tokens;
    }

    /**
     * Convert infix tokens to postfix (RPN)
     * @param {string[]} tokens
     * @returns {string[]}
     */
    function toPostfix(tokens) {
        /** @type {string[]} */
        const output = [];
        /** @type {string[]} */
        const stack = [];

        /** @type {Record<string, { prec: number, rightAssoc?: boolean }>} */
        const ops = {
            "+": { prec: 2 },
            "-": { prec: 2 },
            "*": { prec: 3 },
            "/": { prec: 3 },
            "%": { prec: 3 },
            "^": { prec: 4, rightAssoc: true },
            "neg": { prec: 5, rightAssoc: true }
        };

        const functions = ["sin", "cos", "tan", "log", "ln", "sqrt", "fact", "abs"];

        for (const token of tokens) {
            if (!isNaN(Number(token))) {
                output.push(token);
            } else if (functions.includes(token)) {
                stack.push(token);
            } else if (token in ops) {
                while (stack.length > 0) {
                    const top = stack[stack.length - 1];
                    if (top === "(") break;
                    const topIsFunc = functions.includes(top);
                    const topOp = ops[top];
                    const currOp = ops[token];

                    if (
                        topIsFunc ||
                        (topOp &&
                            (topOp.prec > currOp.prec ||
                                (topOp.prec === currOp.prec && !currOp.rightAssoc)))
                    ) {
                        output.push(/** @type {string} */ (stack.pop()));
                    } else {
                        break;
                    }
                }
                stack.push(token);
            } else if (token === "(") {
                stack.push(token);
            } else if (token === ")") {
                while (stack.length > 0 && stack[stack.length - 1] !== "(") {
                    output.push(/** @type {string} */ (stack.pop()));
                }
                stack.pop(); // discard '('
                if (stack.length > 0 && functions.includes(stack[stack.length - 1])) {
                    output.push(/** @type {string} */ (stack.pop()));
                }
            }
        }

        while (stack.length > 0) {
            output.push(/** @type {string} */ (stack.pop()));
        }

        return output;
    }

    /**
     * Evaluate postfix tokens
     * @param {string[]} postfix
     * @returns {number}
     */
    function evaluatePostfix(postfix) {
        /** @type {number[]} */
        const stack = [];

        for (const token of postfix) {
            if (!isNaN(Number(token))) {
                stack.push(Number(token));
            } else if (token === "neg") {
                const a = stack.pop() ?? 0;
                stack.push(-a);
            } else if (["+", "-", "*", "/", "^", "%"].includes(token)) {
                const b = stack.pop() ?? 0;
                const a = stack.pop() ?? 0;
                switch (token) {
                    case "+": stack.push(a + b); break;
                    case "-": stack.push(a - b); break;
                    case "*": stack.push(a * b); break;
                    case "/": stack.push(b !== 0 ? a / b : NaN); break;
                    case "%": stack.push(a % b); break;
                    case "^": stack.push(Math.pow(a, b)); break;
                }
            } else {
                const a = stack.pop() ?? 0;
                const rad = is_deg ? (a * Math.PI) / 180 : a;
                switch (token) {
                    case "sin": stack.push(Math.sin(rad)); break;
                    case "cos": stack.push(Math.cos(rad)); break;
                    case "tan": stack.push(Math.tan(rad)); break;
                    case "log": stack.push(Math.log10(a)); break;
                    case "ln": stack.push(Math.log(a)); break;
                    case "sqrt": stack.push(Math.sqrt(a)); break;
                    case "abs": stack.push(Math.abs(a)); break;
                    case "fact": stack.push(factorial(a)); break;
                    default: stack.push(a); break;
                }
            }
        }

        return stack.length > 0 ? stack[0] : 0;
    }

    /**
     * Append number or decimal dot
     * @param {string | number} val
     */
    function appendValue(val) {
        const str = String(val);
        if (is_evaluated) {
            expression = "";
            display_number = str === "." ? "0." : str;
            is_evaluated = false;
            return;
        }

        if (display_number === "0" && str !== ".") {
            display_number = str;
        } else {
            if (str === "." && display_number.includes(".")) return;
            display_number += str;
        }
    }

    /**
     * Append basic operator (+, -, ×, ÷, ^, %)
     * @param {string} op
     */
    function appendOperator(op) {
        if (is_evaluated) {
            expression = `${display_number} ${op} `;
            is_evaluated = false;
            display_number = "0";
            return;
        }

        if (display_number !== "0" || expression === "") {
            expression += `${display_number} ${op} `;
            display_number = "0";
        } else if (expression.length > 0) {
            // Replace trailing operator
            expression = expression.trimEnd().replace(/[\+\-\×\÷\^%]$/, op) + " ";
        }
    }

    /**
     * Apply unary scientific function
     * @param {string} fn
     */
    function applyFunction(fn) {
        if (is_evaluated) {
            expression = "";
            is_evaluated = false;
        }

        if (fn === "sqrt") {
            expression += `sqrt(${display_number}) `;
            display_number = "0";
        } else if (fn === "sqr") {
            expression += `(${display_number} ^ 2) `;
            display_number = "0";
        } else if (fn === "inv") {
            expression += `(1 / ${display_number}) `;
            display_number = "0";
        } else if (fn === "fact") {
            expression += `fact(${display_number}) `;
            display_number = "0";
        } else if (["sin", "cos", "tan", "log", "ln"].includes(fn)) {
            expression += `${fn}(${display_number}) `;
            display_number = "0";
        }
    }

    /**
     * Append constant (pi, e)
     * @param {string} constant
     */
    function appendConstant(constant) {
        if (is_evaluated) {
            expression = "";
            is_evaluated = false;
        }
        if (constant === "π") {
            display_number = Math.PI.toString();
        } else if (constant === "e") {
            display_number = Math.E.toString();
        }
    }

    /**
     * Append parenthesis
     * @param {string} p
     */
    function appendParenthesis(p) {
        if (p === "(") {
            if (is_evaluated) {
                expression = "";
                is_evaluated = false;
            }
            expression += "( ";
        } else if (p === ")") {
            if (display_number !== "0") {
                expression += `${display_number} ) `;
                display_number = "0";
            } else {
                expression += ") ";
            }
        }
    }

    /**
     * Toggle sign (+/-)
     */
    function toggleSign() {
        if (display_number === "0") return;
        if (display_number.startsWith("-")) {
            display_number = display_number.slice(1);
        } else {
            display_number = "-" + display_number;
        }
    }

    /**
     * Clear all
     */
    function clearAll() {
        expression = "";
        display_number = "0";
        is_evaluated = false;
    }

    /**
     * Backspace last entered character
     */
    function backspace() {
        if (is_evaluated) {
            clearAll();
            return;
        }
        if (display_number.length > 1) {
            display_number = display_number.slice(0, -1);
        } else {
            display_number = "0";
        }
    }

    /**
     * Calculate equals
     */
    function calculate() {
        let fullExpr = expression;
        if (display_number !== "0" || fullExpr === "") {
            fullExpr += display_number;
        }

        fullExpr = fullExpr.trim();
        if (!fullExpr) return;

        try {
            const result = evaluateExpression(fullExpr);
            expression = `${fullExpr} =`;
            if (isNaN(result) || !Number.isFinite(result)) {
                display_number = "Error";
            } else {
                // Round nicely to avoid floating point inaccuracies
                const rounded = Math.round(result * 1e12) / 1e12;
                display_number = rounded.toString();
            }
            is_evaluated = true;
        } catch {
            display_number = "Error";
            is_evaluated = true;
        }
    }

    /**
     * Handle keyboard events
     * @param {KeyboardEvent} e
     */
    function handleKeyDown(e) {
        const { key } = e;

        if (key >= "0" && key <= "9") {
            e.preventDefault();
            appendValue(key);
        } else if (key === ".") {
            e.preventDefault();
            appendValue(".");
        } else if (key === "+") {
            e.preventDefault();
            appendOperator("+");
        } else if (key === "-") {
            e.preventDefault();
            appendOperator("-");
        } else if (key === "*") {
            e.preventDefault();
            appendOperator("×");
        } else if (key === "/") {
            e.preventDefault();
            appendOperator("÷");
        } else if (key === "%") {
            e.preventDefault();
            appendOperator("%");
        } else if (key === "^") {
            e.preventDefault();
            appendOperator("^");
        } else if (key === "(" || key === ")") {
            e.preventDefault();
            appendParenthesis(key);
        } else if (key === "Enter" || key === "=") {
            e.preventDefault();
            calculate();
        } else if (key === "Backspace") {
            e.preventDefault();
            backspace();
        } else if (key === "Escape" || key === "Delete") {
            e.preventDefault();
            clearAll();
        } else if (key.toLowerCase() === "p") {
            e.preventDefault();
            appendConstant("π");
        } else if (key.toLowerCase() === "e") {
            e.preventDefault();
            appendConstant("e");
        } else if (key === "!") {
            e.preventDefault();
            applyFunction("fact");
        }
    }
</script>

<svelte:window onkeydown={handleKeyDown} />

<div class="calculator-container">
    <div class="calculator">
        <!-- Display Header with live operation above result -->
        <div class="display-container">
            <div class="mode-bar">
                <button
                    type="button"
                    class="mode-badge"
                    onclick={() => (is_deg = !is_deg)}
                    title="Click to toggle Degree / Radian"
                >
                    {is_deg ? "DEG" : "RAD"}
                </button>
                <div class="expression-line">{expression || "\u00A0"}</div>
            </div>
            <div class="main-display">{display_number}</div>
        </div>

        <!-- Buttons Grid -->
        <div class="keypad">
            <!-- Scientific Row 1 -->
            <button type="button" class="btn sci" onclick={() => applyFunction("sin")}>sin</button>
            <button type="button" class="btn sci" onclick={() => applyFunction("cos")}>cos</button>
            <button type="button" class="btn sci" onclick={() => applyFunction("tan")}>tan</button>
            <button type="button" class="btn sci" onclick={() => appendConstant("π")}>π</button>
            <button type="button" class="btn sci" onclick={() => appendConstant("e")}>e</button>

            <!-- Scientific Row 2 -->
            <button type="button" class="btn sci" onclick={() => applyFunction("log")}>log</button>
            <button type="button" class="btn sci" onclick={() => applyFunction("ln")}>ln</button>
            <button type="button" class="btn sci" onclick={() => applyFunction("sqrt")}>√</button>
            <button type="button" class="btn sci" onclick={() => applyFunction("sqr")}>x²</button>
            <button type="button" class="btn sci" onclick={() => appendOperator("^")}>xʸ</button>

            <!-- Standard Row 1 -->
            <button type="button" class="btn sci" onclick={() => applyFunction("fact")}>n!</button>
            <button type="button" class="btn sci" onclick={() => appendParenthesis("(")}>(</button>
            <button type="button" class="btn sci" onclick={() => appendParenthesis(")")}>)</button>
            <button type="button" class="btn danger" onclick={clearAll}>C</button>
            <button type="button" class="btn warning" onclick={backspace}>⌫</button>

            <!-- Standard Row 2 -->
            <button type="button" class="btn sci" onclick={() => applyFunction("inv")}>1/x</button>
            <button type="button" class="btn num" onclick={() => appendValue(7)}>7</button>
            <button type="button" class="btn num" onclick={() => appendValue(8)}>8</button>
            <button type="button" class="btn num" onclick={() => appendValue(9)}>9</button>
            <button type="button" class="btn op" onclick={() => appendOperator("÷")}>÷</button>

            <!-- Standard Row 3 -->
            <button type="button" class="btn sci" onclick={() => appendOperator("%")}>%</button>
            <button type="button" class="btn num" onclick={() => appendValue(4)}>4</button>
            <button type="button" class="btn num" onclick={() => appendValue(5)}>5</button>
            <button type="button" class="btn num" onclick={() => appendValue(6)}>6</button>
            <button type="button" class="btn op" onclick={() => appendOperator("×")}>×</button>

            <!-- Standard Row 4 -->
            <button type="button" class="btn sci" onclick={toggleSign}>±</button>
            <button type="button" class="btn num" onclick={() => appendValue(1)}>1</button>
            <button type="button" class="btn num" onclick={() => appendValue(2)}>2</button>
            <button type="button" class="btn num" onclick={() => appendValue(3)}>3</button>
            <button type="button" class="btn op" onclick={() => appendOperator("-")}>−</button>

            <!-- Standard Row 5 -->
            <button
                type="button"
                class="btn sci"
                onclick={() => (is_deg = !is_deg)}
                title="Angle Unit"
            >
                {is_deg ? "DEG" : "RAD"}
            </button>
            <button type="button" class="btn num" onclick={() => appendValue(0)}>0</button>
            <button type="button" class="btn num" onclick={() => appendValue(".")}>.</button>
            <button type="button" class="btn equals" onclick={calculate}>=</button>
            <button type="button" class="btn op" onclick={() => appendOperator("+")}>+</button>
        </div>
    </div>
</div>

<style>
    :global(body) {
        margin: 0;
        font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
        background: #f0f2f5;
        min-height: 100vh;
        display: flex;
        justify-content: center;
        align-items: center;
    }

    .calculator-container {
        display: flex;
        justify-content: center;
        align-items: center;
        padding: 20px;
        width: 100%;
        box-sizing: border-box;
    }

    .calculator {
        background: #575555;
        border-radius: 18px;
        box-shadow: 0 10px 30px rgba(0, 0, 0, 0.12), 0 1px 4px rgba(0, 0, 0, 0.08);
        padding: 20px;
        width: 100%;
        max-width: 440px;
        box-sizing: border-box;
    }

    .display-container {
        background: #1e293b;
        color: #ffffff;
        border-radius: 12px;
        padding: 14px 16px;
        margin-bottom: 16px;
        box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.2);
    }

    .mode-bar {
        display: flex;
        align-items: center;
        justify-content: space-between;
        min-height: 22px;
        margin-bottom: 6px;
    }

    .mode-badge {
        background: #334155;
        color: #38bdf8;
        font-size: 11px;
        font-weight: 700;
        letter-spacing: 0.5px;
        padding: 2px 8px;
        border-radius: 6px;
        border: none;
        cursor: pointer;
        transition: background 0.15s ease;
    }

    .mode-badge:hover {
        background: #475569;
    }

    .expression-line {
        font-size: 14px;
        color: #94a3b8;
        text-align: right;
        overflow-x: auto;
        white-space: nowrap;
        scrollbar-width: none;
        flex: 1;
        margin-left: 10px;
    }

    .expression-line::-webkit-scrollbar {
        display: none;
    }

    .main-display {
        font-size: 32px;
        font-weight: 600;
        text-align: right;
        color: #f8fafc;
        overflow-x: auto;
        white-space: nowrap;
        scrollbar-width: none;
        letter-spacing: 0.5px;
        line-height: 1.2;
    }

    .main-display::-webkit-scrollbar {
        display: none;
    }

    .keypad {
        display: grid;
        grid-template-columns: repeat(5, 1fr);
        gap: 8px;
    }

    .btn {
        padding: 13px 4px;
        font-size: 17px;
        font-weight: 500;
        border: 1px solid transparent;
        border-radius: 10px;
        cursor: pointer;
        user-select: none;
        transition: all 0.12s ease;
        display: flex;
        justify-content: center;
        align-items: center;
    }

    .btn:active {
        transform: scale(0.96);
    }

    .btn.num {
        background: #f8fafc;
        color: #0f172a;
        border-color: #e2e8f0;
    }

    .btn.num:hover {
        background: #e2e8f0;
    }

    .btn.sci {
        background: #e0e7ff;
        color: #3730a3;
        font-size: 15px;
        font-weight: 600;
    }

    .btn.sci:hover {
        background: #c7d2fe;
    }

    .btn.op {
        background: #f97316;
        color: #ffffff;
        font-size: 19px;
        font-weight: 600;
    }

    .btn.op:hover {
        background: #ea580c;
    }

    .btn.danger {
        background: #ef4444;
        color: #ffffff;
        font-weight: 600;
    }

    .btn.danger:hover {
        background: #dc2626;
    }

    .btn.warning {
        background: #f59e0b;
        color: #ffffff;
        font-size: 17px;
    }

    .btn.warning:hover {
        background: #d97706;
    }

    .btn.equals {
        background: #2563eb;
        color: #ffffff;
        font-size: 20px;
        font-weight: 700;
    }

    .btn.equals:hover {
        background: #1d4ed8;
    }
</style>