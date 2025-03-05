
# Sitema-de-sinais-dinamicos-em-kotlin
```kotlin
import godot.*
import godot.annotation.*
import godot.core.*
import kotlin.reflect.KFunction
import kotlin.reflect.full.findAnnotation

@Target(AnnotationTarget.FUNCTION)
annotation class ConnectSignal(val signalName: String, val nodePath: String = "")

data class SignalData(
    val nodePath: String,
    val nodeRef: WeakRef<Node>
)

class SignalManager : Node() {

    private val signalMap = mutableMapOf<String, MutableList<SignalData>>()
    private val annotatedFunctionsMap = mutableMapOf<String, MutableList<Triple<WeakRef<Node>, String, String>>>()

    override fun _ready() {
        // Conectar sinais "node_added" e "node_removed"
        get_tree()?.connect("node_added", this, "_on_tree_entered")
        get_tree()?.connect("node_removed", this, "_on_tree_exited")

        // Atravessar a árvore de nós e chamar _on_tree_entered para cada nó já presente
        get_tree()?.root?.forEachNode { node ->
            _on_tree_entered(node)
        }
    }

    private fun _on_tree_entered(node: Node) {
        val nodePath = node.get_path().to_string()

        // Adicionar sinais ao signalMap
        node.get_signal_list().forEach { signalName ->
            val signalData = SignalData(nodePath, WeakRef(node))
            signalMap.computeIfAbsent(signalName) { mutableListOf() }.add(signalData)

            // Verificar as conexões pendentes de forma assíncrona
            checkPendingConnectionsAsync(signalName, nodePath)
        }

        // Obter funções anotadas
        getAnnotatedFunctions(node)
    }

    private fun _on_tree_exited(node: Node) {
        val nodePath = node.get_path().to_string()

        // Remover sinais do signalMap
        signalMap.forEach { (signalName, signals) ->
            signals.removeIf { it.nodePath == nodePath }
        }

        // Remover funções anotadas associadas a este nó
        annotatedFunctionsMap.forEach { (_, functions) ->
            functions.removeIf { it.first.get_ref() == node }
        }

        // Desconectar os sinais desse nó
        node.get_signal_list().forEach { signalName ->
            annotatedFunctionsMap[signalName]?.forEach { (receiverRef, methodName, targetPath) ->
                val receiverNode = receiverRef.get_ref()
                if (receiverNode != null && (targetPath.isEmpty() || targetPath == nodePath)) {
                    if (node.is_connected(signalName, receiverNode, methodName)) {
                        node.disconnect(signalName, receiverNode, methodName)
                    }
                }
            }
        }
    }

    private suspend fun checkPendingConnectionsAsync(signalName: String, nodePath: String) {
        // A corrotina é suspensa aqui até que as operações assíncronas estejam prontas.
        val pendingConnections = mutableListOf<Triple<WeakRef<Node>, String, String>>()

        // Verificar as conexões pendentes
        annotatedFunctionsMap[signalName]?.forEach { (nodeRef, methodName, targetPath) ->
            val receiverNode = nodeRef.get_ref()
            if (receiverNode != null && (targetPath.isEmpty() || targetPath == nodePath)) {
                pendingConnections.add(Triple(nodeRef, methodName, targetPath))
            }
        }

        // Aguardar para aplicar as conexões depois
        yield() // Suspende a execução e retoma após o fluxo principal

        // Aplicar as conexões pendentes
        _applyPendingConnections(signalName, pendingConnections)
    }

    private fun _applyPendingConnections(signalName: String, pendingConnections: List<Triple<WeakRef<Node>, String, String>>) {
        pendingConnections.forEach { (receiverRef, methodName, targetPath) ->
            connectCachedSignals(signalName)
        }
    }

    private fun connectCachedSignals(signalName: String) {
        signalMap[signalName]?.forEach { signalData ->
            val targetNode = signalData.nodeRef.get_ref()
            if (targetNode != null) {
                annotatedFunctionsMap[signalName]?.forEach { (receiverRef, methodName, targetPath) ->
                    val receiverNode = receiverRef.get_ref()
                    if (receiverNode != null && (targetPath.isEmpty() || targetPath == signalData.nodePath)) {
                        if (!targetNode.is_connected(signalName, receiverNode, methodName)) {
                            targetNode.connect(signalName, receiverNode, methodName)
                        }
                    }
                }
            } else {
                // Remover sinal se o nó não for mais válido
                signalMap[signalName]?.remove(signalData)
            }
        }
    }

    private fun getAnnotatedFunctions(node: Node) {
        node::class.members.forEach { function ->
            val annotation = function.findAnnotation<ConnectSignal>()
            if (annotation != null && function is KFunction<*>) {
                annotatedFunctionsMap.computeIfAbsent(annotation.signalName) { mutableListOf() }
                    .add(Triple(WeakRef(node), function.name, annotation.nodePath))

                if (annotation.nodePath.isNotEmpty() && get_node_or_null(annotation.nodePath) != null) {
                    // Verificar e conectar sinais pendentes
                    connectCachedSignals(annotation.signalName)
                }
            }
        }
    }

    private fun Node.forEachNode(action: (Node) -> Unit) {
        action(this)
        get_children().forEach { child ->
            if (child is Node) {
                child.forEachNode(action)
            }
        }
    }

    // Função para iniciar a corrotina (caso você precise chamá-la manualmente em outros lugares)
    fun startPendingConnections(signalName: String, nodePath: String) {
        // Criar e iniciar a corrotina para verificar conexões
        Engine.get_singleton().create_coroutine {
            checkPendingConnectionsAsync(signalName, nodePath)
        }
    }
}
```
## aqui um exemplo de uso
```kotlin
class GameManager : Node() {
    val scoreUpdated = signal<Int>()
}

class Player : Node() {
    @ConnectSignal("score_updated", "/root/GameManager")
    fun onScoreUpdated(newScore: Int) {
        println("O jogador viu o novo placar: $newScore")
    }
}

```




